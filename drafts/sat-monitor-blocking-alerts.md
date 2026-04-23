Most blocking alerts are useless. "Blocking detected on SQLPROD01" tells you nothing you can act on. You open SSMS, run `sp_who2`, squint at the output, try to trace the chain, and by the time you've identified the head blocker the blocking has either resolved itself or escalated into a production incident.

What you actually need: "Session 55 (CORP\jsmith, DESKTOP-JSMITH7, SSMS) has been blocking 3 sessions for 45 seconds with an uncommitted UPDATE on dbo.Orders. Longest waiter: session 112, waiting 42 seconds for a shared lock." That's an alert you can act on in seconds.

I used an AI agent to build blocking chain alerts that include that level of context. Here's how the conversation went.

## The Initial Prompt

```text
Write a T-SQL monitoring script for SQL Server that detects blocking chains
and sends an HTML-formatted Database Mail alert. Requirements:
1. Find the head blocker (the session at the root of the chain)
2. Show all sessions blocked directly or transitively by the head blocker
3. Include for each session: session_id, login_name, host_name, program_name,
   wait_type, wait_duration_ms, and the SQL text being executed
4. Include the head blocker's SQL text and whether it's actively executing
   or sleeping
5. Only alert when the head blocker has been blocking for more than 30 seconds
6. Format the alert as an HTML email with a table showing the full chain

Use sys.dm_os_waiting_tasks, sys.dm_exec_requests, sys.dm_exec_sessions,
and sys.dm_exec_sql_text. Use ANSI joins, square brackets, lowercase
datatypes, /* */ comments, and semicolons.
```

## What the Agent Built

The first version was functional but flat — it found blocking pairs from `sys.dm_os_waiting_tasks` but didn't trace the full chain. If session A blocked B, and B blocked C, it showed two separate rows without connecting them. I pushed back:

```text
This only shows direct blocking pairs. I need the full chain traced from
the head blocker. Use a recursive CTE to walk the blocking chain from the
root (where blocking_session_id is not itself blocked by anyone) through
all levels. Add a column showing the chain depth level.
```

The agent rewrote the core query with a recursive CTE anchored on sessions that are blocking others but not themselves blocked. That's the correct approach — the head blocker is a session that appears as a `blocking_session_id` in `sys.dm_os_waiting_tasks` but does not appear as a `session_id` in the same DMV.

## Adding Escalation Logic

A 30-second blocking event and a 5-minute blocking event are different urgency levels. I asked the agent to add tiered alerting:

```text
Add severity levels to the alert:
- WARNING: head blocker has been blocking for 30-119 seconds
- CRITICAL: head blocker has been blocking for 120+ seconds

Change the email subject line to include the severity level and the
server name. For CRITICAL alerts, also CC the on-call manager
distribution list.
```

The agent added a `CASE` expression that set the severity based on the maximum `wait_duration_ms` in the chain, and used conditional logic to set the `@copy_recipients` parameter on `sp_send_dbmail`. Clean implementation.

## What I Validated and Changed

I tested by opening an explicit transaction in SSMS on a dev instance, then running a blocking query from another session. The recursive CTE correctly traced a 3-level chain. A few things I adjusted:

- **The head blocker's SQL text was missing when the session was sleeping.** `sys.dm_exec_requests` only has rows for sessions actively executing a request. For sleeping sessions (the most dangerous blockers — they finished their query but didn't commit), I needed to pull from `sys.dm_exec_connections` using `most_recent_sql_handle` instead. The agent added this as a `COALESCE` between the two sources.
- **System sessions appeared as false head blockers.** Latch waits on system sessions occasionally showed up as blocking. I added `s.[is_user_process] = 1` to filter those out.
- **The chain depth was off by one.** The anchor of the recursive CTE was level 0 (the head blocker), but the email displayed it as level 1. Minor, but confusing during triage.

## The Final Solution

```sql
/* Blocking Chain Monitor with Tiered Alerting */
/* Schedule via SQL Server Agent every 1-2 minutes */
SET NOCOUNT ON;

DECLARE @WarningThresholdMs int = 30000;   /* 30 seconds */
DECLARE @CriticalThresholdMs int = 120000; /* 2 minutes */
DECLARE @ProfileName nvarchar(128) = N'DBA Alerts';
DECLARE @Recipients nvarchar(256) = N'dba-team@example.com';
DECLARE @CriticalCC nvarchar(256) = N'oncall-mgr@example.com';

;WITH [BlockingPairs] AS (
    SELECT
        wt.[session_id] AS [blocked_session_id]
       ,wt.[blocking_session_id]
       ,wt.[wait_type]
       ,wt.[wait_duration_ms]
    FROM sys.dm_os_waiting_tasks AS wt
    WHERE wt.[blocking_session_id] IS NOT NULL
      AND wt.[session_id] <> wt.[blocking_session_id]
)
,[HeadBlockers] AS (
    /* Sessions that block others but are not themselves blocked */
    SELECT DISTINCT bp.[blocking_session_id] AS [session_id]
    FROM [BlockingPairs] AS bp
    WHERE NOT EXISTS (
        SELECT 1
        FROM [BlockingPairs] AS bp2
        WHERE bp2.[blocked_session_id] = bp.[blocking_session_id]
    )
)
,[BlockingChain] AS (
    /* Anchor: head blockers */
    SELECT
        hb.[session_id]
       ,CONVERT(int, NULL) AS [blocking_session_id]
       ,CONVERT(nvarchar(128), N'HEAD BLOCKER') AS [wait_type]
       ,CONVERT(bigint, 0) AS [wait_duration_ms]
       ,0 AS [chain_depth]
    FROM [HeadBlockers] AS hb

    UNION ALL

    /* Recursive: blocked sessions */
    SELECT
        bp.[blocked_session_id]
       ,bp.[blocking_session_id]
       ,bp.[wait_type]
       ,bp.[wait_duration_ms]
       ,bc.[chain_depth] + 1
    FROM [BlockingChain] AS bc
    INNER JOIN [BlockingPairs] AS bp
        ON bp.[blocking_session_id] = bc.[session_id]
    WHERE bc.[chain_depth] < 20 /* safety limit */
)
SELECT
    bc.[chain_depth]
   ,bc.[session_id]
   ,bc.[blocking_session_id]
   ,bc.[wait_type]
   ,bc.[wait_duration_ms]
   ,s.[login_name]
   ,s.[host_name]
   ,s.[program_name]
   ,s.[status] AS [session_status]
   ,COALESCE(sqt_req.[text], sqt_conn.[text]) AS [sql_text]
INTO #BlockingData
FROM [BlockingChain] AS bc
INNER JOIN sys.dm_exec_sessions AS s
    ON s.[session_id] = bc.[session_id]
LEFT JOIN sys.dm_exec_requests AS r
    ON r.[session_id] = bc.[session_id]
LEFT JOIN sys.dm_exec_connections AS c
    ON c.[session_id] = bc.[session_id]
OUTER APPLY sys.dm_exec_sql_text(r.[sql_handle]) AS sqt_req
OUTER APPLY sys.dm_exec_sql_text(c.[most_recent_sql_handle]) AS sqt_conn
WHERE s.[is_user_process] = 1
OPTION (MAXRECURSION 20);

/* Only alert if blocking exceeds the warning threshold */
IF EXISTS (
    SELECT 1 FROM #BlockingData
    WHERE [wait_duration_ms] >= @WarningThresholdMs
)
BEGIN
    DECLARE @MaxWaitMs bigint;
    DECLARE @Severity nvarchar(10);
    DECLARE @CC nvarchar(256);
    DECLARE @Subject nvarchar(256);
    DECLARE @HtmlBody nvarchar(max);

    SELECT @MaxWaitMs = MAX([wait_duration_ms]) FROM #BlockingData;

    SET @Severity = CASE
        WHEN @MaxWaitMs >= @CriticalThresholdMs THEN N'CRITICAL'
        ELSE N'WARNING'
    END;

    SET @CC = CASE
        WHEN @MaxWaitMs >= @CriticalThresholdMs THEN @CriticalCC
        ELSE NULL
    END;

    SET @Subject = @Severity + N': Blocking chain detected on '
        + @@SERVERNAME + N' (max wait '
        + CONVERT(nvarchar(20), @MaxWaitMs / 1000) + N's)';

    SET @HtmlBody = N'<html><body>'
        + N'<h2>' + @Severity + N' — Blocking Chain on '
        + @@SERVERNAME + N'</h2>'
        + N'<table border="1" cellpadding="4" cellspacing="0"'
        + N' style="border-collapse:collapse; font-family:Consolas,monospace;'
        + N' font-size:12px;">'
        + N'<tr style="background:#333; color:#fff;">'
        + N'<th>Depth</th><th>Session</th><th>Blocked By</th>'
        + N'<th>Wait Type</th><th>Wait (sec)</th><th>Status</th>'
        + N'<th>Login</th><th>Host</th><th>Program</th>'
        + N'<th>SQL Text</th></tr>';

    SELECT @HtmlBody = @HtmlBody
        + N'<tr' + CASE
            WHEN [chain_depth] = 0
            THEN N' style="background:#fee;"'
            ELSE N''
          END + N'>'
        + N'<td>' + CONVERT(nvarchar(5), [chain_depth]) + N'</td>'
        + N'<td>' + CONVERT(nvarchar(10), [session_id]) + N'</td>'
        + N'<td>' + ISNULL(CONVERT(nvarchar(10), [blocking_session_id]),
            N'—') + N'</td>'
        + N'<td>' + ISNULL([wait_type], N'') + N'</td>'
        + N'<td>' + CONVERT(nvarchar(20), [wait_duration_ms] / 1000)
            + N'</td>'
        + N'<td>' + ISNULL([session_status], N'') + N'</td>'
        + N'<td>' + ISNULL([login_name], N'') + N'</td>'
        + N'<td>' + ISNULL([host_name], N'') + N'</td>'
        + N'<td>' + ISNULL([program_name], N'') + N'</td>'
        + N'<td>' + ISNULL(LEFT([sql_text], 300), N'(unavailable)')
            + N'</td>'
        + N'</tr>'
    FROM #BlockingData
    ORDER BY [chain_depth], [wait_duration_ms] DESC;

    SET @HtmlBody = @HtmlBody + N'</table>'
        + N'<p>Head blocker rows are highlighted. '
        + N'Review sleeping sessions with open transactions first — '
        + N'they will not resolve on their own.</p></body></html>';

    EXEC msdb.dbo.[sp_send_dbmail]
        @profile_name = @ProfileName
       ,@recipients = @Recipients
       ,@copy_recipients = @CC
       ,@subject = @Subject
       ,@body = @HtmlBody
       ,@body_format = N'HTML';
END;

DROP TABLE IF EXISTS #BlockingData;
```

## Why Context-Rich Alerts Matter

The difference between "blocking detected" and a full chain table with SQL text, login names, and wait durations is the difference between triage taking 30 seconds and triage taking 5 minutes. During a production blocking event, those minutes compound — every second the head blocker holds locks, more sessions pile up behind it.

The escalation logic also prevents alert fatigue. Transient 5-second blocks during normal workload contention never fire the alert. The 30-second warning gives you early notice. The 2-minute critical gets management attention. Your inbox isn't full of noise, so when an alert does arrive, you actually read it.

## Try This Yourself

1. Deploy the script on a test instance. Open two SSMS windows — run `BEGIN TRAN; UPDATE dbo.SomeTable SET col = col WHERE 1=0;` in one, then `SELECT * FROM dbo.SomeTable` in the other.
2. Wait 30 seconds, then run the monitoring query manually to see the chain.
3. Iterate with the agent: ask it to add the estimated number of rows locked by the head blocker, or to include the database name and object name from `sys.dm_tran_locks`.
4. Tune the thresholds for your environment — OLTP systems may want 15-second warnings; data warehouse workloads may tolerate 5-minute thresholds.
5. Schedule as an Agent job every 60–90 seconds. Don't go below 30 seconds — you'll overwhelm Database Mail during real incidents.

For the complete approach to building custom monitoring queries and tuning alert thresholds, see [Post 13: Building Custom Monitoring Queries and Alerts](/ai-for-dbas/building-custom-monitoring/). For deeper analysis of blocking chains, wait stats, and deadlocks, see [Post 8: Wait Stats, Deadlocks, and Blocking Chains](/ai-for-dbas/wait-stats-deadlocks-blocking/).

---

*Part of the [ALTER DBA ADD AGENT](/ai-for-dbas/alter-dba-add-agent/) series.*
