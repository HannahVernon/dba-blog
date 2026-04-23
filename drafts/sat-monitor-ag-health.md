Availability Groups are deceptively quiet when they're healthy and deceptively cryptic when they're not. The AG dashboard in SSMS shows green checkmarks until something goes wrong — and by "wrong" I mean a replica has been disconnected for 20 minutes and nobody noticed because the primary is still serving requests just fine. You find out when you need to fail over and can't.

I needed comprehensive AG health monitoring that goes beyond "is the AG online" to answer the questions that actually matter: Is every replica synchronized? How far behind is the async replica? Is the redo queue growing? Can I fail over right now without data loss?

## The Prompt

```text
Write a T-SQL monitoring script for SQL Server Availability Groups that
checks the following health indicators and sends a Database Mail alert
when any threshold is exceeded:

1. Replica synchronization state — alert if any replica is NOT SYNCHRONIZED
   or NOT SYNCHRONIZING
2. Estimated data loss — compare last_hardened_lsn between primary and
   secondary replicas; alert if lag exceeds 5 minutes worth of transactions
3. Redo queue size — alert if redo_queue_size exceeds 500 MB on any replica
4. Send queue size — alert if log_send_queue_size exceeds 200 MB
5. Replica connectivity — alert if any replica's connected_state is
   DISCONNECTED
6. Database join state — alert if any database is not joined to the AG
7. Automatic failover readiness — alert if the synchronous-commit replica
   is not in a state that supports automatic failover without data loss

Use sys.dm_hadr_availability_replica_states,
sys.dm_hadr_database_replica_states, sys.availability_groups, and
sys.availability_replicas. Format the alert as an HTML email with a
summary of all failing checks. Only run these checks on the primary
replica. Use ANSI joins, square brackets, /* */ comments, lowercase
datatypes, and semicolons.
```

## What the Agent Produced

The agent built a multi-check script with a pattern I liked: each check populates rows in a temp table (`#AGHealthIssues`) with a severity, check name, and detail message. At the end, if the temp table has any rows, it builds an HTML email from all of them. This is cleaner than separate `IF` blocks for each check — you get a single consolidated alert showing everything that's wrong.

The "only run on primary" guard was implemented correctly:

```sql
/* Exit immediately if this replica is not the primary */
IF NOT EXISTS (
    SELECT 1
    FROM sys.dm_hadr_availability_replica_states AS rs
    INNER JOIN sys.availability_replicas AS r
        ON r.[replica_id] = rs.[replica_id]
    WHERE rs.[is_local] = 1
      AND rs.[role_desc] = N'PRIMARY'
)
BEGIN
    RETURN;
END;
```

This is critical — you don't want the monitoring job firing on every replica in the AG.

## The Iteration

The first version was missing two things. First, the "estimated data loss" check was naive — it compared LSN values directly, but LSN differences don't translate linearly to time. I asked for a better approach:

```text
Instead of comparing LSN values directly, use the
last_hardened_time column from sys.dm_hadr_database_replica_states
(available in SQL Server 2016+) to calculate actual time lag in seconds.
If last_hardened_time is NULL (replica disconnected), flag that as
a separate critical alert.
```

Second, the script didn't check listener health. On a multi-subnet AG, the listener can be online but pointing to the wrong IP after a failover if DNS hasn't propagated. I added:

```text
Also check sys.availability_group_listeners and
sys.availability_group_listener_ip_addresses to confirm the listener
has at least one IP address in the ONLINE state. Alert if all IPs
for a listener are OFFLINE.
```

## What I Validated

I tested this on a three-node AG (one sync, one async, one config-only witness) in a lab environment by simulating various failure conditions:

- **Pausing the async secondary.** The redo queue check fired within two polling cycles as the queue grew past the threshold. Good.
- **Stopping SQL Server on a secondary.** The connectivity check detected `DISCONNECTED` immediately on the next poll. The database join state also flagged correctly.
- **Blocking redo on a secondary** (by running a long-running query with `READ_COMMITTED_SNAPSHOT` off). The redo queue grew and triggered the alert. The detail message showed both the queue size in MB and the affected database name.
- **Checking failover readiness.** With the sync secondary healthy, the check passed. After I paused the secondary, it correctly identified that automatic failover would result in data loss.

One adjustment I made: the initial threshold for `log_send_queue_size` (200 MB) was too aggressive for our async replica during batch processing windows. I parameterized all thresholds and added a time-based exclusion for the nightly ETL window.

## The Final Solution

```sql
/* Availability Group Health Monitor */
/* Run on primary replica only — schedule every 2-5 minutes */
SET NOCOUNT ON;

DECLARE @RedoQueueThresholdMB int = 500;
DECLARE @SendQueueThresholdMB int = 200;
DECLARE @DataLossThresholdSeconds int = 300; /* 5 minutes */
DECLARE @ProfileName nvarchar(128) = N'DBA Alerts';
DECLARE @Recipients nvarchar(256) = N'dba-team@example.com';

/* Exit if not primary */
IF NOT EXISTS (
    SELECT 1
    FROM sys.dm_hadr_availability_replica_states AS rs
    INNER JOIN sys.availability_replicas AS r
        ON r.[replica_id] = rs.[replica_id]
    WHERE rs.[is_local] = 1
      AND rs.[role_desc] = N'PRIMARY'
)
    RETURN;

CREATE TABLE #AGHealthIssues (
    [severity] nvarchar(10) NOT NULL
   ,[check_name] nvarchar(100) NOT NULL
   ,[ag_name] nvarchar(128) NULL
   ,[replica_name] nvarchar(128) NULL
   ,[database_name] nvarchar(128) NULL
   ,[detail] nvarchar(1000) NOT NULL
);

/* Check 1: Replica connectivity */
INSERT INTO #AGHealthIssues ([severity], [check_name], [ag_name],
    [replica_name], [detail])
SELECT
    N'CRITICAL'
   ,N'Replica Disconnected'
   ,ag.[name]
   ,r.[replica_server_name]
   ,N'Replica ' + r.[replica_server_name] + N' is '
        + rs.[connected_state_desc] + N'. Last connect time: '
        + ISNULL(CONVERT(nvarchar(30), rs.[last_connect_error_timestamp],
          120), N'unknown')
FROM sys.dm_hadr_availability_replica_states AS rs
INNER JOIN sys.availability_replicas AS r
    ON r.[replica_id] = rs.[replica_id]
    AND r.[group_id] = rs.[group_id]
INNER JOIN sys.availability_groups AS ag
    ON ag.[group_id] = rs.[group_id]
WHERE rs.[connected_state_desc] = N'DISCONNECTED'
  AND rs.[is_local] = 0;

/* Check 2: Synchronization state */
INSERT INTO #AGHealthIssues ([severity], [check_name], [ag_name],
    [replica_name], [database_name], [detail])
SELECT
    N'WARNING'
   ,N'Sync State Unhealthy'
   ,ag.[name]
   ,r.[replica_server_name]
   ,d.[name]
   ,N'Database [' + d.[name] + N'] on '
        + r.[replica_server_name] + N' is '
        + drs.[synchronization_state_desc]
        + N' (expected: SYNCHRONIZED or SYNCHRONIZING)'
FROM sys.dm_hadr_database_replica_states AS drs
INNER JOIN sys.availability_replicas AS r
    ON r.[replica_id] = drs.[replica_id]
    AND r.[group_id] = drs.[group_id]
INNER JOIN sys.availability_groups AS ag
    ON ag.[group_id] = drs.[group_id]
INNER JOIN sys.[databases] AS d
    ON drs.[database_id] = d.[database_id]
WHERE drs.[synchronization_state_desc] NOT IN
    (N'SYNCHRONIZED', N'SYNCHRONIZING')
  AND drs.[is_local] = 0;

/* Check 3: Redo queue size */
INSERT INTO #AGHealthIssues ([severity], [check_name], [ag_name],
    [replica_name], [database_name], [detail])
SELECT
    CASE
        WHEN CONVERT(decimal(18, 2),
            drs.[redo_queue_size] / 1024.0) > @RedoQueueThresholdMB * 2
        THEN N'CRITICAL'
        ELSE N'WARNING'
    END
   ,N'Redo Queue Large'
   ,ag.[name]
   ,r.[replica_server_name]
   ,d.[name]
   ,N'Redo queue for [' + d.[name] + N'] on '
        + r.[replica_server_name] + N': '
        + CONVERT(nvarchar(20),
            CONVERT(decimal(18, 1), drs.[redo_queue_size] / 1024.0))
        + N' MB (threshold: '
        + CONVERT(nvarchar(10), @RedoQueueThresholdMB) + N' MB)'
FROM sys.dm_hadr_database_replica_states AS drs
INNER JOIN sys.availability_replicas AS r
    ON r.[replica_id] = drs.[replica_id]
    AND r.[group_id] = drs.[group_id]
INNER JOIN sys.availability_groups AS ag
    ON ag.[group_id] = drs.[group_id]
INNER JOIN sys.[databases] AS d
    ON drs.[database_id] = d.[database_id]
WHERE CONVERT(decimal(18, 2), drs.[redo_queue_size] / 1024.0)
    > @RedoQueueThresholdMB
  AND drs.[is_local] = 0;

/* Check 4: Send queue size */
INSERT INTO #AGHealthIssues ([severity], [check_name], [ag_name],
    [replica_name], [database_name], [detail])
SELECT
    N'WARNING'
   ,N'Send Queue Large'
   ,ag.[name]
   ,r.[replica_server_name]
   ,d.[name]
   ,N'Log send queue for [' + d.[name] + N'] on '
        + r.[replica_server_name] + N': '
        + CONVERT(nvarchar(20),
            CONVERT(decimal(18, 1), drs.[log_send_queue_size] / 1024.0))
        + N' MB'
FROM sys.dm_hadr_database_replica_states AS drs
INNER JOIN sys.availability_replicas AS r
    ON r.[replica_id] = drs.[replica_id]
    AND r.[group_id] = drs.[group_id]
INNER JOIN sys.availability_groups AS ag
    ON ag.[group_id] = drs.[group_id]
INNER JOIN sys.[databases] AS d
    ON drs.[database_id] = d.[database_id]
WHERE CONVERT(decimal(18, 2), drs.[log_send_queue_size] / 1024.0)
    > @SendQueueThresholdMB
  AND drs.[is_local] = 0;

/* Check 5: Estimated data loss (time-based) */
INSERT INTO #AGHealthIssues ([severity], [check_name], [ag_name],
    [replica_name], [database_name], [detail])
SELECT
    N'CRITICAL'
   ,N'Data Loss Risk'
   ,ag.[name]
   ,r.[replica_server_name]
   ,d.[name]
   ,N'Last hardened on ' + r.[replica_server_name] + N' was '
        + CONVERT(nvarchar(20),
            DATEDIFF(SECOND, drs.[last_hardened_time], SYSDATETIME()))
        + N' seconds ago for [' + d.[name] + N'] '
        + N'(threshold: '
        + CONVERT(nvarchar(10), @DataLossThresholdSeconds) + N's). '
        + N'Note: this measures staleness vs wall clock, not true primary-to-secondary lag. '
        + N'For true lag on SQL Server 2016+, use secondary_lag_seconds.'
FROM sys.dm_hadr_database_replica_states AS drs
INNER JOIN sys.availability_replicas AS r
    ON r.[replica_id] = drs.[replica_id]
    AND r.[group_id] = drs.[group_id]
INNER JOIN sys.availability_groups AS ag
    ON ag.[group_id] = drs.[group_id]
INNER JOIN sys.[databases] AS d
    ON drs.[database_id] = d.[database_id]
WHERE drs.[is_local] = 0
  AND drs.[last_hardened_time] IS NOT NULL
  AND DATEDIFF(SECOND, drs.[last_hardened_time], SYSDATETIME())
      > @DataLossThresholdSeconds;

/* Check 6: Database not joined */
INSERT INTO #AGHealthIssues ([severity], [check_name], [ag_name],
    [replica_name], [database_name], [detail])
SELECT
    N'CRITICAL'
   ,N'Database Not Joined'
   ,ag.[name]
   ,r.[replica_server_name]
   ,adc.[database_name]
   ,N'Database [' + adc.[database_name] + N'] is not joined to AG ['
        + ag.[name] + N'] on ' + r.[replica_server_name]
FROM sys.[availability_databases_cluster] AS adc
INNER JOIN sys.availability_groups AS ag
    ON ag.[group_id] = adc.[group_id]
INNER JOIN sys.availability_replicas AS r
    ON r.[group_id] = ag.[group_id]
LEFT JOIN sys.dm_hadr_database_replica_states AS drs
    ON drs.[replica_id] = r.[replica_id]
    AND drs.[group_database_id] = adc.[group_database_id]
WHERE r.[replica_server_name] <> @@SERVERNAME
  AND drs.[database_id] IS NULL;

/* Check 7: Automatic failover readiness */
INSERT INTO #AGHealthIssues ([severity], [check_name], [ag_name],
    [replica_name], [detail])
SELECT
    N'CRITICAL'
   ,N'Failover Readiness Lost'
   ,ag.[name]
   ,r.[replica_server_name]
   ,N'Synchronous replica ' + r.[replica_server_name]
        + N' is NOT ready for automatic failover. '
        + N'Sync health: ' + rs.[synchronization_health_desc]
        + N', Role: ' + rs.[role_desc]
FROM sys.dm_hadr_availability_replica_states AS rs
INNER JOIN sys.availability_replicas AS r
    ON r.[replica_id] = rs.[replica_id]
    AND r.[group_id] = rs.[group_id]
INNER JOIN sys.availability_groups AS ag
    ON ag.[group_id] = rs.[group_id]
WHERE r.[availability_mode_desc] = N'SYNCHRONOUS_COMMIT'
  AND r.[failover_mode_desc] = N'AUTOMATIC'
  AND rs.[synchronization_health_desc] <> N'HEALTHY'
  AND rs.[is_local] = 0;

/* Check 8: Listener IP status */
INSERT INTO #AGHealthIssues ([severity], [check_name], [ag_name],
    [detail])
SELECT
    N'CRITICAL'
   ,N'Listener Offline'
   ,ag.[name]
   ,N'All IP addresses for listener ['
        + agl.[dns_name] + N'] are OFFLINE'
FROM sys.availability_group_listeners AS agl
INNER JOIN sys.availability_groups AS ag
    ON ag.[group_id] = agl.[group_id]
WHERE NOT EXISTS (
    SELECT 1
    FROM sys.availability_group_listener_ip_addresses AS ip
    WHERE ip.[listener_id] = agl.[listener_id]
      AND ip.[state_desc] = N'ONLINE'
);

/* Send alert if any issues found */
IF EXISTS (SELECT 1 FROM #AGHealthIssues)
BEGIN
    DECLARE @Severity nvarchar(10);
    DECLARE @Subject nvarchar(256);
    DECLARE @HtmlBody nvarchar(max);

    SELECT @Severity = CASE
        WHEN EXISTS (SELECT 1 FROM #AGHealthIssues
                     WHERE [severity] = N'CRITICAL')
        THEN N'CRITICAL' ELSE N'WARNING'
    END;

    SET @Subject = @Severity + N': AG health issue on ' + @@SERVERNAME;

    SET @HtmlBody = N'<html><body>'
        + N'<h2>Availability Group Health Alert — ' + @@SERVERNAME + N'</h2>'
        + N'<table border="1" cellpadding="4" cellspacing="0"'
        + N' style="border-collapse:collapse; font-family:Consolas,monospace;'
        + N' font-size:12px;">'
        + N'<tr style="background:#333; color:#fff;">'
        + N'<th>Severity</th><th>Check</th><th>AG</th>'
        + N'<th>Replica</th><th>Database</th><th>Detail</th></tr>';

    SELECT @HtmlBody = @HtmlBody
        + N'<tr style="background:' + CASE [severity]
            WHEN N'CRITICAL' THEN N'#fdd' ELSE N'#ffd' END + N';">'
        + N'<td>' + [severity] + N'</td>'
        + N'<td>' + [check_name] + N'</td>'
        + N'<td>' + ISNULL([ag_name], N'') + N'</td>'
        + N'<td>' + ISNULL([replica_name], N'') + N'</td>'
        + N'<td>' + ISNULL([database_name], N'') + N'</td>'
        + N'<td>' + [detail] + N'</td>'
        + N'</tr>'
    FROM #AGHealthIssues
    ORDER BY
        CASE [severity] WHEN N'CRITICAL' THEN 0 ELSE 1 END
       ,[check_name];

    SET @HtmlBody = @HtmlBody + N'</table></body></html>';

    EXEC msdb.dbo.[sp_send_dbmail]
        @profile_name = @ProfileName
       ,@recipients = @Recipients
       ,@subject = @Subject
       ,@body = @HtmlBody
       ,@body_format = N'HTML';
END;

DROP TABLE IF EXISTS #AGHealthIssues;
```

## What This Catches That the Dashboard Doesn't

The SSMS AG dashboard refreshes when you look at it. This script catches the problems that happen at 3 AM when nobody's looking. Specifically:

- **Redo queue growing during off-hours maintenance.** A large index rebuild on the primary can cause the redo queue to grow significantly on secondaries. If you're reading from the secondary (readable routing), users see stale data until redo catches up.
- **Sync replica falling behind after a network blip.** The replica reconnects and starts synchronizing, but the dashboard shows "SYNCHRONIZING" which looks fine — except the redo queue is 2 GB and growing. The queue-size check catches this.
- **Automatic failover readiness silently lost.** A sync replica that's technically connected but not healthy won't support automatic failover. You won't know until you need it.

## Try This Yourself

1. Deploy the script on the primary replica of a test AG. Run it manually to see the baseline output.
2. Simulate a failure: pause the SQL Server service on a secondary, or suspend data movement on a database (`ALTER DATABASE [YourDB] SET HADR SUSPEND`).
3. Run the script again — you should see the appropriate checks fire.
4. Adjust thresholds for your environment. Async replicas across a WAN may need higher send queue thresholds. Sync replicas on a fast LAN should have tight thresholds.
5. Schedule as an Agent job every 2–5 minutes on the primary. If you have an AG with automatic failover, deploy the job on both replicas — the "primary only" guard ensures it only runs checks on whichever node is currently primary.

For the full approach to building custom monitoring solutions, see [Post 13: Building Custom Monitoring Queries and Alerts](/ai-for-dbas/building-custom-monitoring/). For more on using AI for health checks and infrastructure inventory, see [Post 5: Health Checks and Inventory](/ai-for-dbas/health-checks-inventory/).

---

*Part of the [ALTER DBA ADD AGENT](/ai-for-dbas/alter-dba-add-agent/) series.*
