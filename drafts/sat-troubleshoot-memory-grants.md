Memory grants are one of those things that only become visible when they're already a problem. A query requests 4 GB of workspace memory for a sort or hash join, gets it, uses 12 MB, and holds the rest hostage for the duration of execution. Meanwhile, other queries queue behind `RESOURCE_SEMAPHORE` waits because there's no workspace memory left. Or the opposite: a query gets a tiny grant, runs out of memory mid-execution, and spills to tempdb — turning a 2-second query into a 45-second one.

The data to diagnose both scenarios lives in `sys.dm_exec_query_memory_grants` and `sys.dm_exec_query_stats`. The challenge is interpreting it under pressure and knowing which fix applies to which pattern. This is where an AI agent earns its keep.

## Collecting Memory Grant Data

Start with currently executing queries and their memory grants:

```sql
/* Active memory grants — who's holding memory right now? */
SELECT
    mg.[session_id]
  , mg.[request_time]
  , mg.[required_memory_kb]
  , mg.[requested_memory_kb]
  , mg.[granted_memory_kb]
  , mg.[used_memory_kb]
  , mg.[max_used_memory_kb]
  , mg.[ideal_memory_kb]
  , mg.[queue_id]
  , mg.[wait_time_ms]
  , mg.[is_small]
  , mg.[dop]
  , st.[text] AS [sql_text]
  , qp.[query_plan]
FROM sys.dm_exec_query_memory_grants AS mg
CROSS APPLY sys.dm_exec_sql_text(mg.[sql_handle]) AS st
CROSS APPLY sys.dm_exec_query_plan(mg.[plan_handle]) AS qp
WHERE mg.[granted_memory_kb] IS NOT NULL
ORDER BY mg.[granted_memory_kb] DESC;
```

For queries waiting for grants (the `RESOURCE_SEMAPHORE` case), check for rows where `granted_memory_kb` is `NULL`:

```sql
/* Queries waiting for memory grants */
SELECT
    mg.[session_id]
  , mg.[request_time]
  , mg.[requested_memory_kb]
  , mg.[granted_memory_kb]
  , mg.[wait_time_ms]
  , st.[text] AS [sql_text]
FROM sys.dm_exec_query_memory_grants AS mg
CROSS APPLY sys.dm_exec_sql_text(mg.[sql_handle]) AS st
WHERE mg.[granted_memory_kb] IS NULL
ORDER BY mg.[wait_time_ms] DESC;
```

For historical analysis of spills, Query Store is your best source in SQL Server 2022 and later — it tracks `last_tempdb_space_used` at the plan level. For earlier versions, check `sys.dm_exec_query_stats` columns like `total_spills` (added in SQL Server 2017).

## Feeding the Data to the Agent

Here's the prompt pattern I use. Include actual numbers — the agent can't diagnose "high memory grants" without seeing the data.

```text
I'm troubleshooting memory grant issues on a SQL Server 2022 instance with
128 GB max server memory. Here's the output from sys.dm_exec_query_memory_grants
for currently executing queries:

session | requested_kb | granted_kb  | used_kb   | max_used_kb | wait_ms | dop | query (abbreviated)
--------|-------------|-------------|-----------|-------------|---------|-----|---------------------
112     | 4,194,304   | 4,194,304   | 12,288    | 14,336      | 0       | 8   | SELECT ... ORDER BY ...
198     | 2,097,152   | 2,097,152   | 1,843,200 | 1,843,200   | 0       | 4   | SELECT ... GROUP BY ... HAVING ...
215     | 524,288     | NULL        | NULL      | NULL        | 34,221  | 1   | INSERT INTO #tmp SELECT ...
301     | 262,144     | NULL        | NULL      | NULL        | 28,102  | 1   | SELECT ... JOIN ... ORDER BY ...
87      | 131,072     | 131,072     | 131,072   | 131,072     | 0       | 1   | UPDATE ... FROM ... JOIN ...

Server: RESOURCE_SEMAPHORE waits are the #3 wait type (189,000 ms in
the last hour). Total workspace memory available: ~32 GB.

Analyze these grants and tell me:
1. Which queries have grant-vs-used problems?
2. Why are sessions 215 and 301 waiting for grants?
3. What's causing the RESOURCE_SEMAPHORE waits?
4. What fixes do you recommend for each query?
```

## What the Agent Produces

The agent identifies three distinct patterns in this data:

**Session 112: Massive over-grant.** Requested and received 4 GB but only used 14 MB — a 300:1 ratio. This is almost certainly a cardinality estimation error. The optimizer estimated a huge sort or hash operation and reserved memory for it, but the actual data volume was tiny. This query is holding 4 GB of workspace memory hostage while sessions 215 and 301 wait.

**Session 198: Healthy grant.** Requested 2 GB, used 1.8 GB. This is a legitimate large operation — the grant matches usage. No action needed on this one, though you might ask whether the query *needs* to process that much data.

**Sessions 215 and 301: Grant waiters.** Both are waiting for memory grants (`granted_kb` is `NULL`). They need 512 MB and 256 MB respectively — not unreasonable amounts — but they can't get memory because session 112 is holding 4 GB it isn't using. This is the direct cause of the `RESOURCE_SEMAPHORE` waits.

**Session 87: Spill candidate.** Granted exactly what it's using (128 MB). Check the execution plan for sort or hash spill warnings — if the query originally needed more than 128 MB but was capped, it may be spilling to tempdb.

## What I Validated and Changed

I pulled the execution plan for session 112 and confirmed the agent's suspicion: a sort operator estimated 2.8 million rows but only processed 1,200. The cardinality estimate was off because statistics on the filtered column were stale — last updated 3 weeks ago when the table had very different data distribution.

```sql
/* Update statistics on the problem table */
UPDATE STATISTICS dbo.[TransactionHistory]
    [IX_TransactionHistory_PostDate]
    WITH FULLSCAN;
```

After the statistics update, I forced a recompile of the procedure. The new plan requested 18 MB instead of 4 GB.

For the longer-term fix, I added memory grant feedback guardrails. SQL Server 2022 has [memory grant feedback persistence](https://learn.microsoft.com/en-us/sql/relational-databases/performance/intelligent-query-processing-memory-grant-feedback) (part of Intelligent Query Processing) that learns from execution history and adjusts grants automatically. But for critical queries where I can't wait for the feedback loop, I use query hints:

```sql
/* Cap the memory grant for the transaction history report */
SELECT
    th.[PostDate]
  , th.[AccountID]
  , a.[AccountName]
  , SUM(th.[Amount]) AS [DailyTotal]
FROM dbo.[TransactionHistory] AS th
INNER JOIN dbo.[Accounts] AS a ON a.[AccountID] = th.[AccountID]
WHERE th.[PostDate] >= @StartDate
  AND th.[PostDate] < @EndDate
GROUP BY th.[PostDate], th.[AccountID], a.[AccountName]
ORDER BY th.[PostDate], [DailyTotal] DESC
OPTION (MAX_GRANT_PERCENT = 5);
```

`MAX_GRANT_PERCENT` caps the grant at a percentage of total workspace memory. At 5% of 32 GB, that's ~1.6 GB — still generous, but prevents the 4 GB runaway scenario. I also use `MIN_GRANT_PERCENT` for queries that consistently spill:

```sql
/* Ensure minimum grant for the import batch that always spills */
INSERT INTO dbo.[StagingTable] ([Col1], [Col2], [Col3])
SELECT s.[Col1], s.[Col2], s.[Col3]
FROM dbo.[SourceData] AS s
INNER JOIN dbo.[LookupTable] AS l ON l.[LookupKey] = s.[LookupKey]
OPTION (MIN_GRANT_PERCENT = 3);
```

For broader protection, consider Resource Governor to cap memory grants per workload group — especially useful for separating ad-hoc reporting queries from OLTP workloads:

```sql
/* Resource Governor workload group for reporting */
ALTER WORKLOAD GROUP [ReportingGroup]
WITH
(
    REQUEST_MAX_MEMORY_GRANT_PERCENT = 15
  , MAX_DOP = 4
);
ALTER RESOURCE GOVERNOR RECONFIGURE;
```

## The Diagnostic Checklist

When the agent flags a memory grant problem, here's the validation sequence:

1. **Over-grants** (granted >> used): Check statistics freshness and cardinality estimates. Update statistics, then recompile.
2. **Spills** (used = granted and plan shows spill warnings): The query needs *more* memory. Check if `MIN_GRANT_PERCENT` or updated statistics would help.
3. **RESOURCE_SEMAPHORE waits**: Find the sessions holding the most granted memory. Usually one or two over-granted queries are starving everything else.
4. **Persistent issues**: Enable memory grant feedback (on by default in SQL Server 2022 with Query Store enabled) or use Resource Governor.

## Try This Yourself

1. Run the active memory grants query above during your peak workload.
2. Look for sessions with a high granted-to-used ratio — anything above 5:1 is worth investigating.
3. Feed the output to your AI agent with your server's memory configuration.
4. Check statistics freshness on the tables involved in over-granted queries.
5. For chronic over-granters, test `MAX_GRANT_PERCENT` in a non-production environment first — setting it too low will cause spills.

Memory grants are a balancing act. The agent helps you spot the imbalance faster, but the fix depends on whether the root cause is stale statistics, bad cardinality estimates, or genuinely large data operations that need Resource Governor limits.

For more on AI-assisted performance diagnostics, see [Post 8: Wait Stats, Deadlocks, and Blocking Chains](/ai-for-dbas/wait-stats-deadlocks-blocking/). For building automated monitoring that catches these patterns before they become incidents, see [Post 13: Building Custom Monitoring](/ai-for-dbas/building-custom-monitoring/).

---

*Part of the [ALTER DBA ADD AGENT](/ai-for-dbas/alter-dba-add-agent/) series.*
