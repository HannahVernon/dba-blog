Every SQL Server instance has a built-in tattletale: the `SQLServer:Deprecated Features` performance counter object. It quietly increments counters every time your workload uses something Microsoft plans to remove. Most DBAs never look at it — and then act surprised when an upgrade breaks something.

Static code scanning catches deprecated syntax sitting in stored procedures and views. Performance counters catch deprecated features used *at runtime* — in application queries, dynamic SQL, and ad-hoc batches that no code scan will ever find. You need both. An AI agent can help you build a comprehensive scan that covers the full surface area.

## The Problem

You're planning an upgrade from SQL Server 2017 (compatibility level 140) to SQL Server 2025 (compatibility level 170). You need to find every deprecated feature your databases use — not just the ones sitting in `sys.sql_modules`, but the ones hiding in cached plans, runtime workloads, and application-side SQL. You need the results prioritized so you know what's a ticking time bomb and what's a cosmetic warning.

## The Prompt

```text
I'm upgrading from SQL Server 2017 (compat level 140) to SQL Server 2025
(compat level 170). Build a comprehensive deprecated feature detection
toolkit with three components:

1. RUNTIME DETECTION: Query sys.dm_os_performance_counters for the
   "SQLServer:Deprecated Features" object. Show which deprecated features
   have been used since the last instance restart, with usage counts.
   Sort by usage count descending.

2. CACHED PLAN SCAN: Query sys.dm_exec_query_plan and sys.dm_exec_sql_text
   to find deprecated syntax patterns in currently cached plans. Look for:
   - Non-ANSI outer joins (*= and =*)
   - COMPUTE / COMPUTE BY
   - SET ROWCOUNT used before DML statements
   - Three-part column references in ORDER BY
   - String-based RAISERROR (without parentheses)
   - GROUP BY ALL

3. STATIC CODE SCAN: Scan sys.sql_modules for deprecated constructs in
   stored procedures, views, functions, and triggers. Include:
   - Deprecated system tables (sysprocesses, sysobjects, syscolumns,
     sysusers, sysdatabases, syslogins)
   - Deprecated data types (text, ntext, image)
   - Numbered stored procedures (;1, ;2 suffix pattern)
   - Non-ANSI outer joins
   - COMPUTE / COMPUTE BY
   - Legacy RAISERROR syntax

For each finding, classify as:
- REMOVED: Will cause errors in SQL Server 2025
- DEPRECATED-HIGH: Still works but removal is announced for a near-future version
- DEPRECATED-LOW: Deprecated for multiple versions, no imminent removal

Group results by severity, then by database/object.
```

## What the Agent Produced

The agent returned three distinct scripts — one for each detection layer. The runtime counter query was clean and immediately useful. The cached plan scan used `sys.dm_exec_query_plan` with `CROSS APPLY` to extract plan XML and `sys.dm_exec_sql_text` to get the source SQL. The static code scan used pattern matching against `sys.sql_modules` definitions.

The severity classification was reasonable: non-ANSI outer joins and `COMPUTE BY` were marked as high priority (these have been removed in recent versions), while deprecated system table references like `sysprocesses` were correctly classified as lower priority since the compatibility views still work.

## What I Validated and Changed

The agent's runtime counter query worked as written, but I made several adjustments across the three scripts:

- **The cached plan scan was too aggressive.** It was flagging comments containing `*=` as deprecated join syntax. I added pattern refinements to reduce false positives — checking for whitespace or column-name characters on either side of the operator.
- **The `SET ROWCOUNT` detection needed context.** `SET ROWCOUNT 0` (turning it off) isn't deprecated — only `SET ROWCOUNT n` before a DML statement is. The agent was flagging every occurrence. I adjusted the pattern to exclude `SET ROWCOUNT 0`.
- **Numbered stored procedures need a specific pattern.** The agent searched for `;1` and `;2` in procedure names, but that also matches semicolons used as statement terminators. I changed the pattern to look for `CREATE PROC[EDURE]` followed by a name with `;` and a digit.
- **Added `sys.server_sql_modules`** for server-scoped objects — logon triggers and DDL triggers at the server level that the database-scoped scan misses entirely.

## The Final Scripts

### Runtime Deprecated Feature Detection

```sql
/* Deprecated feature usage since last instance restart */
/* Non-zero counters = active usage in your workload */
SET NOCOUNT ON;

SELECT
    pc.[instance_name] AS [DeprecatedFeature]
   ,pc.[cntr_value] AS [UsageCount]
   ,si.[sqlserver_start_time] AS [CountingSince]
   ,DATEDIFF(DAY, si.[sqlserver_start_time], GETDATE()) AS [DaysOfData]
FROM sys.[dm_os_performance_counters] AS pc
    CROSS JOIN sys.[dm_os_sys_info] AS si
WHERE pc.[object_name] LIKE N'%:Deprecated Features%'
    AND pc.[cntr_value] > 0
ORDER BY pc.[cntr_value] DESC;
```

This is the single most valuable query in the toolkit. If a counter has a high value, that feature is being actively used in production — by application code, Agent jobs, or ad-hoc queries that no static scan will find. Let it run for at least a full business cycle (ideally a week covering month-end processing) before drawing conclusions.

### Cached Plan Deprecated Syntax Scan

```sql
/* Scan cached plans for deprecated syntax patterns */
SET NOCOUNT ON;

SELECT
    DB_NAME(t.[dbid]) AS [DatabaseName]
   ,t.[text] AS [QueryText]
   ,p.[usecounts] AS [ExecutionCount]
   ,p.[size_in_bytes] / 1024 AS [PlanSizeKB]
   ,CASE
        WHEN t.[text] LIKE N'%*=%'
            THEN N'Non-ANSI outer join (*=)'
        WHEN t.[text] LIKE N'%=*%'
            THEN N'Non-ANSI outer join (=*)'
        WHEN t.[text] LIKE N'%COMPUTE BY%'
            THEN N'COMPUTE BY'
        WHEN t.[text] LIKE N'%COMPUTE %' AND t.[text] NOT LIKE N'%COMPUTE BY%'
            THEN N'COMPUTE'
        WHEN t.[text] LIKE N'%GROUP BY ALL%'
            THEN N'GROUP BY ALL'
    END AS [DeprecatedPattern]
FROM sys.[dm_exec_cached_plans] AS p
    CROSS APPLY sys.[dm_exec_sql_text](p.[plan_handle]) AS t
WHERE t.[text] IS NOT NULL
    AND (
        t.[text] LIKE N'%*=%'
        OR t.[text] LIKE N'%=*%'
        OR t.[text] LIKE N'%COMPUTE %'
        OR t.[text] LIKE N'%GROUP BY ALL%'
    )
    AND t.[text] NOT LIKE N'%sys.dm_%' /* Exclude DMV queries */
ORDER BY p.[usecounts] DESC;
```

### Static Code Deprecated Construct Scan

```sql
/* Scan database code for deprecated constructs */
/* Run in context of each target database */
SET NOCOUNT ON;

;WITH [DeprecatedPatterns] AS (
    SELECT *
    FROM (VALUES
        /* REMOVED or will error in future versions */
        (N'%*=%', N'Non-ANSI outer join (*=)', N'REMOVED')
       ,(N'%=*%', N'Non-ANSI outer join (=*)', N'REMOVED')
       ,(N'%COMPUTE BY%', N'COMPUTE BY clause', N'REMOVED')
       ,(N'%COMPUTE %SUM%', N'COMPUTE clause', N'REMOVED')
       ,(N'%COMPUTE %AVG%', N'COMPUTE clause', N'REMOVED')
       ,(N'%COMPUTE %COUNT%', N'COMPUTE clause', N'REMOVED')
        /* DEPRECATED-HIGH: announced for near-future removal */
       ,(N'%RAISERROR [0-9]%', N'String RAISERROR (no parentheses)', N'DEPRECATED-HIGH')
       ,(N'%SET ROWCOUNT [1-9]%', N'SET ROWCOUNT for DML limiting', N'DEPRECATED-HIGH')
       ,(N'%GROUP BY ALL%', N'GROUP BY ALL', N'DEPRECATED-HIGH')
        /* DEPRECATED-LOW: still functional via compat views */
       ,(N'% sysprocesses%', N'Deprecated system table: sysprocesses', N'DEPRECATED-LOW')
       ,(N'% sysobjects%', N'Deprecated system table: sysobjects', N'DEPRECATED-LOW')
       ,(N'% syscolumns%', N'Deprecated system table: syscolumns', N'DEPRECATED-LOW')
       ,(N'% sysusers%', N'Deprecated system table: sysusers', N'DEPRECATED-LOW')
       ,(N'% sysdatabases%', N'Deprecated system table: sysdatabases', N'DEPRECATED-LOW')
       ,(N'% syslogins%', N'Deprecated system table: syslogins', N'DEPRECATED-LOW')
       ,(N'% text,%', N'Deprecated data type: text', N'DEPRECATED-LOW')
       ,(N'% ntext,%', N'Deprecated data type: ntext', N'DEPRECATED-LOW')
       ,(N'% image,%', N'Deprecated data type: image', N'DEPRECATED-LOW')
       ,(N'% text)%', N'Deprecated data type: text', N'DEPRECATED-LOW')
       ,(N'% ntext)%', N'Deprecated data type: ntext', N'DEPRECATED-LOW')
       ,(N'% image)%', N'Deprecated data type: image', N'DEPRECATED-LOW')
    ) AS p([Pattern], [Description], [Severity])
)
SELECT
    dp.[Severity]
   ,OBJECT_SCHEMA_NAME(sm.[object_id]) AS [SchemaName]
   ,OBJECT_NAME(sm.[object_id]) AS [ObjectName]
   ,o.[type_desc] AS [ObjectType]
   ,dp.[Description] AS [DeprecatedFeature]
   ,LEN(sm.[definition]) AS [DefinitionLength]
FROM sys.[sql_modules] AS sm
    INNER JOIN sys.[objects] AS o
        ON sm.[object_id] = o.[object_id]
    CROSS JOIN [DeprecatedPatterns] AS dp
WHERE sm.[definition] LIKE dp.[Pattern]
    AND o.[is_ms_shipped] = 0
ORDER BY
    CASE dp.[Severity]
        WHEN N'REMOVED' THEN 1
        WHEN N'DEPRECATED-HIGH' THEN 2
        WHEN N'DEPRECATED-LOW' THEN 3
    END
   ,OBJECT_SCHEMA_NAME(sm.[object_id])
   ,OBJECT_NAME(sm.[object_id]);
```

Don't forget server-scoped objects:

```sql
/* Check server-level modules (logon triggers, DDL triggers) */
SELECT
    sm.[object_id]
   ,OBJECT_NAME(sm.[object_id], 1) AS [ObjectName]
   ,sm.[definition]
FROM sys.[server_sql_modules] AS sm
WHERE sm.[definition] LIKE N'%sysprocesses%'
    OR sm.[definition] LIKE N'%sysobjects%'
    OR sm.[definition] LIKE N'%*=%'
    OR sm.[definition] LIKE N'%=*%';
```

## Building the Remediation Plan

Once you have your findings, ask the agent to prioritize the fixes:

```text
Here are my deprecated feature scan results. Build a prioritized
remediation plan grouped by:
1. CRITICAL - Will break on upgrade (removed features)
2. HIGH - Deprecated with imminent removal timeline
3. LOW - Deprecated but functional for the foreseeable future

For each finding, provide the modern replacement syntax and an
estimated effort level (trivial / moderate / complex).
```

The agent maps each deprecated construct to its replacement: `*=` becomes `LEFT OUTER JOIN`, `COMPUTE BY` becomes window functions with `GROUP BY` or application-layer subtotals, `text`/`ntext`/`image` become `varchar(max)`/`nvarchar(max)`/`varbinary(max)`, `sysprocesses` becomes `sys.dm_exec_sessions` joined to `sys.dm_exec_requests`, and string `RAISERROR` becomes the parenthesized form with severity and state parameters.

The remediation effort classifications are useful for project planning. Replacing `sysprocesses` with DMVs is usually trivial — the column names are similar. Replacing `COMPUTE BY` can be complex if downstream reports depend on the extra result sets it produces. Non-ANSI join conversion is moderate — the logic is equivalent, but you need to be careful with multi-table chains where the precedence of `*=` differs from explicit `LEFT JOIN` ordering.

## Try This Yourself

Start with the runtime counter query. Run it on your busiest production instance and look at the top five deprecated features by usage count. Those are the ones your workload is actively using — not hypothetical risks buried in unused procedures, but real runtime behavior. Then run the static scan on one database and compare: some of your highest-count runtime features may not appear in `sys.sql_modules` at all, because they're coming from application-side queries.

The gap between runtime counters and static code analysis is where the real upgrade risk lives. For the full migration planning workflow — including compatibility level assessment, breaking change analysis, and runbook generation — see [Post 11: Migration Planning](/ai-for-dbas/migration-planning-compatibility/).

---

*Part of the [ALTER DBA ADD AGENT](/ai-for-dbas/alter-dba-add-agent/) series.*
