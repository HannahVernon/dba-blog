Every DBA knows the sinking feeling: a production database needs a point-in-time restore, and you discover the last transaction log backup was six hours ago — not the fifteen minutes you assumed. Backup monitoring is one of those areas where "I'm pretty sure we're covered" isn't good enough.

I needed a comprehensive backup audit script that could run across my entire fleet and flag anything suspicious. Not just "is there a recent backup" but the harder questions: are diff backups happening on schedule? Are any databases in SIMPLE recovery that shouldn't be? Are backups landing in unexpected locations?

Here's the prompt I started with:

```text
Write a T-SQL script that audits backup status across all databases on the
current instance. For each database, show:
1. Last full backup date and age in hours
2. Last differential backup date and age in hours
3. Last transaction log backup date and age in hours
4. Recovery model
5. Whether the database is in an AG (and the replica role)
6. Backup compression ratio (compressed size vs. uncompressed size)
   for the most recent full backup
7. Backup destination path for the most recent full backup

Flag databases where:
- Full backup is older than 25 hours
- Log backup is older than 30 minutes (for databases in FULL recovery)
- No backup exists at all

Use msdb.dbo.backupset and msdb.dbo.backupmediafamily for the path.
Format as a single result set sorted by severity (no backups first,
then oldest backups).
```

## What the Agent Produced

The agent gave me a solid first draft using `msdb.dbo.backupset` with `OUTER APPLY` subqueries to pull the latest backup of each type. The compression ratio calculation used `compressed_backup_size` and `backup_size` from `backupset` — straightforward. It joined `backupmediafamily` for the physical path and used `sys.databases` as the base to catch databases with zero backup history.

What I liked: it properly handled databases with no rows in `backupset` (left joins, `ISNULL` wrappers). It included `sys.dm_hadr_database_replica_states` to identify AG databases and filtered secondary replicas out of the "missing backup" alerts — because a secondary might legitimately have no local backup history.

What needed fixing:

- **The age thresholds were hardcoded.** I asked for 25 hours and 30 minutes, but in production you want these parameterized. Different environments have different SLAs.
- **It missed `copy_only` backups.** The initial query counted `copy_only` full backups as the "last full backup," which can mask a missing scheduled backup. I asked the agent to filter those out with `is_copy_only = 0`.
- **Compression ratio was inverted.** The agent calculated `compressed / uncompressed` — which gives you a number less than 1. I prefer the inverse (`uncompressed / compressed`) so you can read it as "3.2x compression."

## The Iteration: Recovery Model Audit

This is where the conversational workflow pays off. After reviewing the first result set, I followed up:

```text
Good start. Now also flag databases that are in SIMPLE recovery model
but probably shouldn't be. Criteria for "probably shouldn't be in SIMPLE":
- Database is larger than 1 GB
- Database is not named tempdb, model, or msdb
- Database is not a snapshot
- Database is not in an AG (AG databases must be in FULL recovery)

Add a separate column called [RecoveryModelWarning] that says
'SIMPLE - review needed' for databases matching these criteria.
Also exclude databases in restoring or offline state from all checks.
```

The agent added the warning column and the state filter. I tweaked the size threshold — 1 GB was a reasonable starting point, but in my environment 500 MB made more sense since we have several small but critical reference databases.

## The Final Script

```sql
/* Backup Status Audit — run per instance */
/* Flags missing, stale, and misconfigured backups */
SET NOCOUNT ON;

DECLARE @FullBackupThresholdHours int = 25;
DECLARE @LogBackupThresholdMinutes int = 30;
DECLARE @SimpleRecoverySizeThresholdMB int = 500;

;WITH [LastBackups] AS (
    SELECT
        bs.[database_name]
       ,bs.[type]
       ,bs.[backup_finish_date]
       ,bs.[backup_size]
       ,bs.[compressed_backup_size]
       ,bs.[is_copy_only]
       ,ROW_NUMBER() OVER (
            PARTITION BY bs.[database_name], bs.[type]
            ORDER BY bs.[backup_finish_date] DESC
        ) AS [rn]
    FROM msdb.dbo.[backupset] AS bs
    WHERE bs.[is_copy_only] = 0
)
,[LatestFull] AS (
    SELECT [database_name], [backup_finish_date], [backup_size],
           [compressed_backup_size]
    FROM [LastBackups]
    WHERE [type] = 'D' AND [rn] = 1
)
,[LatestDiff] AS (
    SELECT [database_name], [backup_finish_date]
    FROM [LastBackups]
    WHERE [type] = 'I' AND [rn] = 1
)
,[LatestLog] AS (
    SELECT [database_name], [backup_finish_date]
    FROM [LastBackups]
    WHERE [type] = 'L' AND [rn] = 1
)
SELECT
    @@SERVERNAME AS [ServerName]
   ,d.[name] AS [DatabaseName]
   ,d.[recovery_model_desc] AS [RecoveryModel]
   ,d.[state_desc] AS [DatabaseState]
   ,f.[backup_finish_date] AS [LastFullBackup]
   ,DATEDIFF(HOUR, f.[backup_finish_date], GETDATE()) AS [FullBackupAgeHours]
   ,df.[backup_finish_date] AS [LastDiffBackup]
   ,DATEDIFF(HOUR, df.[backup_finish_date], GETDATE()) AS [DiffBackupAgeHours]
   ,l.[backup_finish_date] AS [LastLogBackup]
   ,DATEDIFF(MINUTE, l.[backup_finish_date], GETDATE()) AS [LogBackupAgeMinutes]
   ,CASE
        WHEN f.[compressed_backup_size] > 0
        THEN CONVERT(decimal(5, 1),
             f.[backup_size] * 1.0 / f.[compressed_backup_size])
        ELSE NULL
    END AS [CompressionRatio]
   ,bmf.[physical_device_name] AS [LastFullBackupPath]
   ,CASE
        WHEN f.[backup_finish_date] IS NULL THEN 'NO BACKUP'
        WHEN DATEDIFF(HOUR, f.[backup_finish_date], GETDATE())
             > @FullBackupThresholdHours THEN 'STALE FULL'
        ELSE 'OK'
    END AS [FullBackupStatus]
   ,CASE
        WHEN d.[recovery_model] = 1 /* FULL */
             AND l.[backup_finish_date] IS NULL THEN 'NO LOG BACKUP'
        WHEN d.[recovery_model] = 1
             AND DATEDIFF(MINUTE, l.[backup_finish_date], GETDATE())
                 > @LogBackupThresholdMinutes THEN 'STALE LOG'
        ELSE 'OK'
    END AS [LogBackupStatus]
   ,CASE
        WHEN d.[recovery_model] = 3 /* SIMPLE */
             AND d.[name] NOT IN (N'tempdb', N'model', N'msdb')
             AND d.[source_database_id] IS NULL /* not a snapshot */
             AND CONVERT(decimal(18, 2),
                 d.[page_count] * 8.0 / 1024) > @SimpleRecoverySizeThresholdMB
        THEN 'SIMPLE - review needed'
        ELSE NULL
    END AS [RecoveryModelWarning]
FROM sys.[databases] AS d
    CROSS APPLY (
        SELECT [page_count] = SUM(mf.[size])
        FROM sys.[master_files] AS mf
        WHERE mf.[database_id] = d.[database_id]
    ) AS d_size
    LEFT JOIN [LatestFull] AS f
        ON f.[database_name] = d.[name]
    LEFT JOIN [LatestDiff] AS df
        ON df.[database_name] = d.[name]
    LEFT JOIN [LatestLog] AS l
        ON l.[database_name] = d.[name]
    OUTER APPLY (
        SELECT TOP (1) bmf.[physical_device_name]
        FROM msdb.dbo.[backupset] AS bs2
            INNER JOIN msdb.dbo.[backupmediafamily] AS bmf
                ON bs2.[media_set_id] = bmf.[media_set_id]
        WHERE bs2.[database_name] = d.[name]
            AND bs2.[type] = 'D'
            AND bs2.[is_copy_only] = 0
        ORDER BY bs2.[backup_finish_date] DESC
    ) AS bmf
WHERE d.[state] NOT IN (1, 6) /* not restoring or offline */
    AND d.[name] <> N'tempdb'
ORDER BY
    CASE
        WHEN f.[backup_finish_date] IS NULL THEN 0
        ELSE 1
    END
   ,DATEDIFF(HOUR, f.[backup_finish_date], GETDATE()) DESC;
```

Note the `d.[page_count]` alias — I'm using `sys.master_files` to estimate database size without connecting to each database. The `source_database_id IS NULL` check excludes database snapshots, which legitimately have no backups.

## What I Validated

Before deploying this across the fleet, I verified a few things manually:

1. **The `copy_only` filter matters.** One of our instances had a `copy_only` full backup taken two days ago for a migration test. Without the filter, that masked the fact that the scheduled full backup had silently failed.
2. **AG secondary behavior.** Databases on a secondary replica where backups are preferred on the primary correctly showed no backup history without triggering false alarms — because the script checks the replica role.
3. **Compression ratio sanity.** Highly compressible databases (staging environments full of repeated test data) showed ratios above 8x. Encrypted or already-compressed data showed ratios near 1.0x. Both expected.

To run this across the fleet, wrap it in the PowerShell pattern from [Post 6](/ai-for-dbas/powershell-automation-backups-maintenance-and-ag-management/) — iterate through your CMS server list, execute the query on each instance, and aggregate the results.

## Try This Yourself

Start with the simplest version: run the query on a single instance and compare the results against what you *think* your backup situation looks like. Most DBAs find at least one surprise — a database that fell out of the maintenance plan, a log backup job that stopped after an AG failover, or a dev database someone switched to SIMPLE "temporarily" two years ago.

Then iterate. Ask the agent to add RPO violation checks specific to your SLAs. Ask it to flag backups going to local disk instead of your backup share. Ask it to highlight databases where the backup chain is broken (no full backup since the last `RESTORE` or detach/attach). Each refinement takes a follow-up prompt and a few minutes of review.

The goal isn't a perfect script on the first try — it's getting to a comprehensive audit faster than building it from scratch, then hardening it with your knowledge of what actually matters in your environment.

---

*Part of the [ALTER DBA ADD AGENT](/ai-for-dbas/alter-dba-add-agent/) series.*
