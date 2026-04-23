Every DBA has a story about the transaction that ate the log drive. Someone opened an explicit transaction in SSMS at 9 AM, ran an UPDATE, got pulled into a meeting, and by 11 AM the transaction log had grown to fill the disk. The query finished in milliseconds — the *transaction* was open for two hours, holding locks, preventing log truncation, and silently making everything worse.

Long-running queries get caught by monitoring tools. Long-running *transactions* often don't — because the session looks idle. The monitoring dashboard shows green; the log drive is screaming.

I used an AI agent to build a monitoring solution that catches these before they become incidents. Here's the prompt I started with:

```text
Write a T-SQL monitoring query that detects open transactions older than
5 minutes on SQL Server. Include:
1. Session ID, login name, host name, program name
2. Transaction name and start time
3. Duration in minutes
4. Whether the session is currently executing or idle (sleeping)
5. The most recent SQL text from sys.dm_exec_connections
6. Number of locks held by the session
7. Transaction log space used in MB (from sys.dm_tran_database_transactions)

Join sys.dm_tran_active_transactions to sys.dm_tran_session_transactions
to sys.dm_exec_sessions. Use ANSI joins, square-bracket identifiers,
lowercase datatypes, and /* */ comments.
```

## What the Agent Produced

The first draft was structurally sound — it correctly joined `sys.dm_tran_active_transactions` → `sys.dm_tran_session_transactions` → `sys.dm_exec_sessions` and pulled `most_recent_sql_handle` from `sys.dm_exec_connections`. It included a `LEFT JOIN` to `sys.dm_exec_requests` to detect whether the session was actively running or idle, and aggregated lock counts from `sys.dm_tran_locks`.

Two things needed fixing. First, the log space calculation was pulling from `sys.dm_tran_database_transactions` without aggregating — if a transaction spans multiple databases, you get duplicate rows. I asked the agent to fix that:

```text
The log space query produces duplicate rows when a transaction touches
multiple databases. Aggregate the log space used across all databases
per transaction_id, and also show which databases are involved as a
comma-separated list.
```

Second, I realized the initial query would fire on every open transaction — including perfectly normal ones. Backup jobs, index rebuilds, and DBCC operations all hold transactions open for extended periods. I followed up:

```text
Exclude transactions where the session's program_name contains
'SQLAgent - TSQL JobStep' or 'DatabaseMail' or starts with
'DatabaseBackup'. Also add a parameter for the threshold in minutes
so I can tune it without editing the query.
```

The agent updated both correctly. It used `s.[program_name] NOT LIKE` patterns with appropriate wildcards and added a `DECLARE @ThresholdMinutes int` variable at the top.

## What I Validated and Changed

I tested against a development instance where I deliberately opened an explicit transaction in SSMS. The query caught it within one polling cycle. Good. Then I ran it on production during a maintenance window and confirmed:

- Index rebuild transactions showed up until I added the `program_name` exclusion
- The `most_recent_sql_handle` text was the *last batch* on the connection, not the statement that opened the transaction — an important distinction I added as a comment in the script
- Log space numbers matched what `DBCC SQLPERF(LOGSPACE)` showed, confirming the aggregation was correct

I also added the query text from `sys.dm_exec_sql_text` via the connection's `most_recent_sql_handle`, since knowing *what* the session last ran is critical for deciding whether to kill it or wait.

## The Final Solution

```sql
/* Long-Running Open Transaction Monitor */
/* Detects transactions open longer than @ThresholdMinutes */
/* Schedule via SQL Server Agent every 5 minutes */
SET NOCOUNT ON;

DECLARE @ThresholdMinutes int = 5;
DECLARE @ProfileName nvarchar(128) = N'DBA Alerts';
DECLARE @Recipients nvarchar(256) = N'dba-team@example.com';

IF EXISTS
(
    SELECT 1
    FROM sys.dm_tran_active_transactions AS at
    INNER JOIN sys.dm_tran_session_transactions AS st
        ON st.[transaction_id] = at.[transaction_id]
    INNER JOIN sys.dm_exec_sessions AS s
        ON s.[session_id] = st.[session_id]
    WHERE at.[transaction_begin_time] < DATEADD(MINUTE, -@ThresholdMinutes, SYSDATETIME())
      AND s.[is_user_process] = 1
      AND s.[program_name] NOT LIKE N'SQLAgent - TSQL JobStep%'
      AND s.[program_name] NOT LIKE N'DatabaseMail%'
      AND s.[program_name] NOT LIKE N'DatabaseBackup%'
)
BEGIN
    DECLARE @HtmlBody nvarchar(max);

    SET @HtmlBody = N'<html><body>'
        + N'<h2>Long-Running Open Transactions on ' + @@SERVERNAME + N'</h2>'
        + N'<table border="1" cellpadding="4" cellspacing="0">'
        + N'<tr><th>Session ID</th><th>Login</th><th>Host</th>'
        + N'<th>Program</th><th>Status</th><th>Duration (min)</th>'
        + N'<th>Log Used (MB)</th><th>Lock Count</th>'
        + N'<th>Last SQL Text</th></tr>';

    SELECT @HtmlBody = @HtmlBody
        + N'<tr>'
        + N'<td>' + CONVERT(nvarchar(10), s.[session_id]) + N'</td>'
        + N'<td>' + ISNULL(s.[login_name], N'') + N'</td>'
        + N'<td>' + ISNULL(s.[host_name], N'') + N'</td>'
        + N'<td>' + ISNULL(s.[program_name], N'') + N'</td>'
        + N'<td>' + ISNULL(s.[status], N'') + N'</td>'
        + N'<td>' + CONVERT(nvarchar(20),
            DATEDIFF(MINUTE, at.[transaction_begin_time], SYSDATETIME()))
            + N'</td>'
        + N'<td>' + ISNULL(CONVERT(nvarchar(20),
            CONVERT(decimal(18, 2), log_agg.[log_bytes_used] / 1048576.0)),
            N'0') + N'</td>'
        + N'<td>' + CONVERT(nvarchar(10),
            ISNULL(lck.[lock_count], 0)) + N'</td>'
        + N'<td>' + ISNULL(LEFT(sqt.[text], 200), N'(unavailable)') + N'</td>'
        + N'</tr>'
    FROM sys.dm_tran_active_transactions AS at
    INNER JOIN sys.dm_tran_session_transactions AS st
        ON st.[transaction_id] = at.[transaction_id]
    INNER JOIN sys.dm_exec_sessions AS s
        ON s.[session_id] = st.[session_id]
    LEFT JOIN sys.dm_exec_connections AS c
        ON c.[session_id] = s.[session_id]
    OUTER APPLY sys.dm_exec_sql_text(c.[most_recent_sql_handle]) AS sqt
    LEFT JOIN sys.dm_exec_requests AS r
        ON r.[session_id] = s.[session_id]
    LEFT JOIN (
        /* Aggregate log space across all databases per transaction */
        SELECT
            dt.[transaction_id]
           ,SUM(dt.[database_transaction_log_bytes_used]) AS [log_bytes_used]
        FROM sys.dm_tran_database_transactions AS dt
        GROUP BY dt.[transaction_id]
    ) AS log_agg ON log_agg.[transaction_id] = at.[transaction_id]
    LEFT JOIN (
        SELECT
            [request_session_id]
           ,COUNT(*) AS [lock_count]
        FROM sys.dm_tran_locks
        GROUP BY [request_session_id]
    ) AS lck ON lck.[request_session_id] = s.[session_id]
    WHERE at.[transaction_begin_time] < DATEADD(MINUTE, -@ThresholdMinutes, SYSDATETIME())
      AND s.[is_user_process] = 1
      AND s.[program_name] NOT LIKE N'SQLAgent - TSQL JobStep%'
      AND s.[program_name] NOT LIKE N'DatabaseMail%'
      AND s.[program_name] NOT LIKE N'DatabaseBackup%'
    ORDER BY at.[transaction_begin_time];

    SET @HtmlBody = @HtmlBody + N'</table>'
        + N'<p><em>Note: Last SQL Text shows the most recent batch on the '
        + N'connection, not necessarily the statement that opened the '
        + N'transaction.</em></p></body></html>';

    EXEC msdb.dbo.[sp_send_dbmail]
        @profile_name = @ProfileName
       ,@recipients = @Recipients
       ,@subject = N'Long-Running Open Transaction Alert'
       ,@body = @HtmlBody
       ,@body_format = N'HTML';
END;
```

The key insight: **session status matters more than duration alone.** A sleeping session with an open transaction for 10 minutes is almost certainly an abandoned SSMS window. An executing session with a transaction open for 10 minutes might be a legitimate batch job. The `status` column in the alert gives you that context instantly.

## Scheduling It

Create a SQL Server Agent job with a single T-SQL step containing the script above. Schedule it every 5 minutes. For the threshold, 5 minutes is a reasonable starting point for OLTP workloads — adjust upward if you have legitimate long-running batch processes that hold transactions open. If you need more granular exclusions, add specific login names or host names to the `WHERE` clause.

Don't be tempted to check every 30 seconds — you'll overwhelm Database Mail during a real incident, and a transaction that's only been open for 30 seconds rarely needs intervention. Five minutes gives you reasonable detection time without alert flood.

## Try This Yourself

1. Open SSMS, connect to a test instance, and run `BEGIN TRAN; SELECT 1;` — don't commit.
2. Run the monitoring query manually. Your session should appear with a sleeping status.
3. Iterate with the agent: ask it to add the databases involved in the transaction, or to flag transactions holding more than 1000 locks.
4. Deploy as an Agent job and tune the threshold over a week — watch what fires and adjust.
5. The common tuning pattern: start strict (5 minutes), then loosen for specific programs or logins that legitimately hold long transactions.

For the full pattern on building custom monitoring queries and tuning alert thresholds, see [Post 13: Building Custom Monitoring Queries and Alerts](/ai-for-dbas/building-custom-monitoring/). For how this fits into a broader monitoring strategy alongside purpose-built tools, see [Post 12: AI-Native Monitoring](/ai-for-dbas/ai-native-monitoring/).

---

*Part of the [ALTER DBA ADD AGENT](/ai-for-dbas/alter-dba-add-agent/) series.*
