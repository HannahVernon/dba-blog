Compatibility level is one of those settings that's easy to forget about. You migrate a database from SQL Server 2016 to 2022, everything works, and you move on. Six months later you wonder why Query Store's new intelligent query processing features aren't kicking in — and then you notice the database is still running at compatibility level 130.

It happens constantly. A database gets restored, attached, or migrated, and the compatibility level stays wherever it was. Nobody gets an alert. Nothing breaks. You just quietly miss out on years of query optimizer improvements.

## Why Compatibility Level Matters

This isn't just a cosmetic number. Compatibility level controls which query optimizer behaviors are active. Here's what you're leaving on the table at older levels:

| Compat Level | SQL Server Version | Key Features Locked Behind This Level |
|---|---|---|
| 100 | SQL Server 2008 | — |
| 110 | SQL Server 2012 | — |
| 120 | SQL Server 2014 | New cardinality estimator |
| 130 | SQL Server 2016 | Batch mode for columnstore, Query Store basics |
| 140 | SQL Server 2017 | Adaptive joins, interleaved execution |
| 150 | SQL Server 2019 | Batch mode on rowstore, scalar UDF inlining, table variable deferred compilation |
| 160 | SQL Server 2022 | Parameter-sensitive plan optimization, cardinality estimation feedback, DOP feedback, optimized plan forcing |
| 170 | SQL Server 2025 | Latest optimizer enhancements |

A database at compat level 130 on a SQL Server 2022 instance is using a 2016-era optimizer. It won't get adaptive joins, batch mode on rowstore, or any of the intelligent QP features. The engine it's running on is capable of much more — but the compat level is holding it back.

## The Prompt

```text
Write a T-SQL script that audits database compatibility levels on the
current instance. For each database, show:
1. Database name
2. Current compatibility level
3. The SQL Server version that compat level corresponds to
4. The instance's current compatibility level (what the DB *should* be at)
5. How many levels behind the database is
6. Recovery model
7. Database size in MB
8. Whether the database is in an AG

Flag databases where the compat level is lower than the instance's
default. Exclude system databases except msdb (which can legitimately
be at an older compat level after an in-place upgrade).
Classify each database as:
- 'Current' if at the instance default
- 'Review' if 1 level behind
- 'Outdated' if 2 or more levels behind
```

## What the Agent Produced

The agent generated a clean query using `sys.databases` with a `VALUES`-based mapping table. It pulled the instance's default compat level from `SERVERPROPERTY('ProductMajorVersion')` and compared each database's `compatibility_level` against it.

The classification logic was straightforward: calculate the gap between instance default and database compat level, then bucket into categories. The agent correctly used the compat level *number* gap (160 vs. 130 = 30, which is 3 major versions) rather than counting versions — since compat levels increment by 10.

What I adjusted:

- **The agent excluded all system databases.** I wanted `msdb` included because after an in-place upgrade, `msdb` sometimes stays at the old level and needs manual updating.
- **Added Query Store status per database.** If a database is at an older compat level *and* Query Store isn't enabled, that's a double miss — you're not getting optimizer improvements, and you have no baseline data to validate the upgrade.
- **Added `create_date` from `sys.databases`.** This gives context for why a database might be at an old level — a database created in 2016 and never touched probably has compat level 130.

## The Final Script

```sql
/* Compatibility Level Audit */
/* Identifies databases not using current optimizer features */
SET NOCOUNT ON;

DECLARE @InstanceCompatLevel int = CONVERT(int,
    SERVERPROPERTY('ProductMajorVersion')) * 10;

;WITH [CompatMap] AS (
    SELECT *
    FROM (VALUES
        (100, N'SQL Server 2008')
       ,(110, N'SQL Server 2012')
       ,(120, N'SQL Server 2014')
       ,(130, N'SQL Server 2016')
       ,(140, N'SQL Server 2017')
       ,(150, N'SQL Server 2019')
       ,(160, N'SQL Server 2022')
       ,(170, N'SQL Server 2025')
    ) AS c([CompatLevel], [VersionName])
)
,[DbSizes] AS (
    SELECT
        mf.[database_id]
       ,CONVERT(decimal(18, 2),
            SUM(mf.[size]) * 8.0 / 1024) AS [SizeMB]
    FROM sys.[master_files] AS mf
    GROUP BY mf.[database_id]
)
SELECT
    d.[name] AS [DatabaseName]
   ,d.[compatibility_level] AS [CurrentCompatLevel]
   ,cm.[VersionName] AS [CompatLevelVersion]
   ,@InstanceCompatLevel AS [InstanceDefaultLevel]
   ,icm.[VersionName] AS [InstanceVersion]
   ,((@InstanceCompatLevel - d.[compatibility_level]) / 10)
        AS [VersionsBehind]
   ,CASE
        WHEN d.[compatibility_level] = @InstanceCompatLevel THEN 'Current'
        WHEN d.[compatibility_level] = @InstanceCompatLevel - 10 THEN 'Review'
        WHEN d.[compatibility_level] < @InstanceCompatLevel - 10 THEN 'Outdated'
        ELSE 'Current'
    END AS [Status]
   ,d.[recovery_model_desc] AS [RecoveryModel]
   ,ds.[SizeMB]
   ,d.[create_date] AS [CreateDate]
   ,CASE
        WHEN d.[is_query_store_on] = 1 THEN 'Enabled'
        ELSE 'Disabled'
    END AS [QueryStoreStatus]
   ,CASE
        WHEN drs.[replica_id] IS NOT NULL THEN 'Yes'
        ELSE 'No'
    END AS [InAvailabilityGroup]
FROM sys.[databases] AS d
    INNER JOIN [CompatMap] AS cm
        ON d.[compatibility_level] = cm.[CompatLevel]
    INNER JOIN [CompatMap] AS icm
        ON @InstanceCompatLevel = icm.[CompatLevel]
    LEFT JOIN [DbSizes] AS ds
        ON d.[database_id] = ds.[database_id]
    LEFT JOIN sys.[dm_hadr_database_replica_states] AS drs
        ON d.[database_id] = drs.[database_id]
        AND drs.[is_local] = 1
WHERE d.[name] NOT IN (N'master', N'tempdb', N'model')
    AND d.[state] = 0 /* ONLINE */
ORDER BY
    CASE
        WHEN d.[compatibility_level] = @InstanceCompatLevel THEN 2
        WHEN d.[compatibility_level] = @InstanceCompatLevel - 10 THEN 1
        ELSE 0
    END
   ,d.[compatibility_level]
   ,d.[name];
```

## Risk Assessment: Which Databases Can You Upgrade?

Not every database can simply have its compatibility level bumped. The compat level change is instant and doesn't require downtime, but it changes optimizer behavior — and that can cause plan regressions.

Here's the risk framework I use:

**Low risk — upgrade without hesitation:**
- Databases at compat level 150 on a level 160 instance (one version behind, optimizer changes are incremental)
- Small databases under 10 GB with simple workloads
- Dev and test databases (change them first to validate)

**Medium risk — upgrade with Query Store monitoring:**
1. Enable Query Store if it's not already on
2. Collect a baseline for at least a week under normal workload
3. Change the compat level
4. Monitor Query Store for regressed queries
5. Use plan forcing to pin good plans if regressions appear
6. If things go badly, `ALTER DATABASE [YourDb] SET COMPATIBILITY_LEVEL = <old_level>` reverts instantly

**High risk — requires testing first:**
- Databases jumping multiple compat levels (e.g., 120 → 160)
- Large OLTP databases with complex query patterns
- Databases using undocumented or deprecated features
- Databases with known parameter sniffing problems (the new optimizer might change which plans get cached)
- Third-party vendor databases (check vendor support matrix first)

The agent can help you generate the Query Store monitoring queries for tracking plan regressions after a compat level change. That's a natural follow-up prompt: "Write a query that compares query performance before and after a compatibility level change using Query Store."

## The Fleet View

When you run this across your fleet (see [Post 5](/ai-for-dbas/health-checks-inventory/) for the CMS/PowerShell pattern), the aggregate results tell a story. In one environment I audited, 23 out of 89 databases were at least two compat levels behind. Eight of them were still at level 120 — missing seven years of optimizer improvements on instances fully capable of running at level 160.

The compat level upgrade project that followed recovered measurable query performance on several high-volume databases, just by enabling features the engine already supported. No code changes, no schema changes — just flipping a setting with proper monitoring in place.

For databases that need deeper analysis before changing compatibility levels — especially those involved in migration projects — see [Post 11](/ai-for-dbas/migration-planning-compatibility/) for how the agent can help assess compatibility risks.

## Try This Yourself

Run the audit on one instance. Look at the `Status` and `VersionsBehind` columns. If you see databases marked `Outdated` that are two or more versions behind, pick the lowest-risk one (smallest, simplest workload) and walk through the upgrade process: enable Query Store, baseline for a week, bump the compat level, monitor for regressions. The agent can generate every script you need along the way.

Most DBAs who run this audit for the first time discover they've been leaving query optimizer improvements on the table for years — not because they didn't know about them, but because nobody audited compat levels after the last migration.

---

*Part of the [ALTER DBA ADD AGENT](/ai-for-dbas/alter-dba-add-agent/) series.*
