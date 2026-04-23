You've seen this before. A stored procedure runs in 200 milliseconds for most users, then one day it takes 45 seconds. Nothing changed — same procedure, same server, same indexes. Clear the plan cache for that specific procedure with `sp_recompile` and it's fast again. Until it isn't. (And yes, I've seen people restart SQL Server or run `DBCC FREEPROCCACHE` as a "fix" — both are instance-wide hammers that destabilize everything else. Don't do that.)

That's parameter sniffing. SQL Server compiles a query plan optimized for the *first* set of parameter values that triggers compilation. If those values are atypical — a sales rep who handles 3 accounts running a procedure that's later called for a sales rep who handles 30,000 — the cached plan is terrible for the majority of calls.

**A note for SQL Server 2022+ users:** If your database is at compatibility level 160 or higher, check whether [Parameter Sensitive Plan (PSP) optimization](https://learn.microsoft.com/en-us/sql/relational-databases/performance/parameter-sensitive-plan-optimization) is already producing multiple plan variants for your query. PSP may already mitigate the issue without any manual intervention. Query `sys.query_store_plan` for plans with `plan_type = 2` (PSP dispatcher plans) to see if PSP is active for your procedure.

Diagnosing parameter sniffing is straightforward once you know what to look for, but it requires correlating symptoms, plan cache data, and execution statistics. This is a good use case for an AI agent — not because the analysis is beyond a DBA's skills, but because gathering and correlating the evidence is tedious and the agent is thorough.

## Describing the Symptoms

Start by describing what you're seeing. The agent can't query your server directly (unless you've configured an MCP connection), but it can guide you through the diagnostic process and analyze the artifacts you provide.

```text
I have a stored procedure [dbo].[usp_GetOrdersByRep] in the OrderProcessing
database on SQL Server 2022. It usually runs in under 500ms but intermittently
takes 30-60 seconds. It accepts a @SalesRepID int parameter. The procedure
queries dbo.Orders (approximately 12 million rows) joined to dbo.OrderDetails
and dbo.Customers. There's a nonclustered index on Orders(SalesRepID) that
includes OrderDate and CustomerID.

Here's the procedure definition:
[paste procedure text]

Here's the execution plan XML from a slow execution:
[paste XML]

Diagnose the parameter sniffing issue. Identify the sniffed parameter values,
explain why the plan is suboptimal for other values, and recommend fixes with
tradeoffs for each.
```

Providing the execution plan XML is critical. Without it, the agent can only guess. With it, the agent can read the compiled parameter values, the estimated vs. actual row counts, and the specific operators that are performing poorly.

## What the Agent Identifies

From the plan XML, the agent typically finds:

1. **The sniffed parameter values.** In the plan XML, these are in the `ParameterList` element — `ParameterCompiledValue` shows what value was used when the plan was compiled, and `ParameterRuntimeValue` shows what was passed during the slow execution. The agent reads both and highlights the mismatch.

2. **The row count discrepancy.** The plan was compiled expecting 47 rows (the sniffed rep's order count) but the runtime rep has 185,000 rows. The optimizer chose a nested loop join with a key lookup — perfect for 47 rows, catastrophic for 185,000.

3. **The operator that's suffering.** Usually a key lookup or nested loop that should be a hash join or clustered index scan at higher row counts. The agent identifies the operator, explains why it was chosen, and describes what the optimizer would have chosen with accurate cardinality estimates.

Here's a condensed version of what the agent might explain:

```text
The plan was compiled with @SalesRepID = 4287, which returns 47 rows.
The optimizer chose:
  - Index Seek on IX_Orders_SalesRepID (47 estimated rows)
  - Key Lookup to the clustered index for remaining columns
  - Nested Loop Join to OrderDetails

When @SalesRepID = 12 (a national account rep with 185,000 orders), the
Key Lookup executes 185,000 times. Each lookup is a random I/O to the
clustered index. The plan degrades from 200ms to 45+ seconds.

A plan compiled for @SalesRepID = 12 would use:
  - Clustered Index Scan on Orders (filtered)
  - Hash Join to OrderDetails
  - This is slower for 47 rows but dramatically faster for 185,000.
```

## The Recommended Fixes

The agent will typically suggest several approaches. Here's how I evaluate each one:

### OPTION(RECOMPILE)

```sql
SELECT /* ... */
FROM [dbo].[Orders] AS o
INNER JOIN [dbo].[OrderDetails] AS od
    ON o.[OrderID] = od.[OrderID]
WHERE o.[SalesRepID] = @SalesRepID
OPTION (RECOMPILE);
```

**Tradeoff:** Every execution gets a plan optimized for its actual parameter values. Compilation overhead is real — typically 5-50ms per execution — but for a procedure that runs a few hundred times per day, it's negligible. For a procedure that runs 50,000 times per day, the cumulative compilation CPU may be significant. The agent correctly flags this distinction.

**When I use it:** Most of the time. For procedures that run less than a few thousand times per day, `RECOMPILE` is the simplest, most maintainable fix.

### OPTIMIZE FOR

```sql
SELECT /* ... */
FROM [dbo].[Orders] AS o
INNER JOIN [dbo].[OrderDetails] AS od
    ON o.[OrderID] = od.[OrderID]
WHERE o.[SalesRepID] = @SalesRepID
OPTION (OPTIMIZE FOR (@SalesRepID = 500));
```

**Tradeoff:** You're choosing a "typical" value that produces a plan that works reasonably well for most inputs. It's a compromise — not optimal for any single value, but acceptable for most. The risk is that the "typical" value changes over time as data distribution shifts, and no one remembers to update the hint.

`OPTIMIZE FOR UNKNOWN` is the variant that uses average statistics instead of a specific value. It avoids the stale-hint problem but may produce a mediocre plan for everyone instead of a great plan for most.

**When I use it:** When `RECOMPILE` is too expensive (high-frequency procedures) and I can identify a genuinely representative parameter value.

### Dynamic SQL

```sql
DECLARE @SQL nvarchar(max);
SET @SQL = N'
    SELECT /* ... */
    FROM [dbo].[Orders] AS o
    INNER JOIN [dbo].[OrderDetails] AS od
        ON o.[OrderID] = od.[OrderID]
    WHERE o.[SalesRepID] = @pSalesRepID;';

EXEC sys.sp_executesql @SQL
    , N'@pSalesRepID int'
    , @pSalesRepID = @SalesRepID;
```

**Tradeoff:** `sp_executesql` still caches and sniffs parameters — it does *not* inherently behave like `OPTIMIZE FOR UNKNOWN`. Where dynamic SQL helps is when you generate different query text for different predicate combinations (e.g., omitting a `WHERE` clause entirely when a parameter is `NULL`). Each distinct SQL text gets its own cached plan, so the optimizer sees different queries rather than one query with a sniffed value for an unused predicate. The added complexity (harder to read, debug, and maintain) is the cost. Permissions can also be tricky — the dynamic SQL executes in its own security context.

**When I use it:** When the procedure has multiple parameters that interact (not just one problematic parameter) and I need per-parameter plan variation. Also useful when I'm building conditional WHERE clauses anyway.

### Plan Guides

Plan guides force a specific plan shape without modifying the procedure code. The agent can generate the `sp_create_plan_guide` call, but I rarely use this approach — plan guides are fragile, easy to forget about, and silently stop applying when someone modifies the procedure text.

**When I use it:** Almost never. Sometimes as an emergency production fix when I can't modify the procedure immediately.

## Try This Yourself

If you suspect parameter sniffing in your environment:

1. **Capture the plan XML** from a slow execution. Use Query Store, an Extended Events session, or `SET STATISTICS XML ON`.
2. **Identify the compiled vs. runtime parameter values** in the `ParameterList` element.
3. **Paste the procedure definition and plan XML** into the agent with the diagnostic prompt above.
4. **Ask for fix recommendations with tradeoffs** — don't just accept the first suggestion. Ask the agent to compare approaches for your specific workload pattern.

The agent won't replace your judgment on which fix is right for your environment — that depends on execution frequency, data distribution, maintenance practices, and deployment constraints that only you know. But it will give you a thorough analysis to base that judgment on.

For the broader T-SQL workflow, see [Writing T-SQL with an AI Partner](/ai-for-dbas/writing-tsql-with-ai-partner/). For using AI to analyze wait stats, deadlocks, and other performance problems, see [Wait Stats, Deadlocks, and Blocking Analysis](/ai-for-dbas/wait-stats-deadlocks-blocking/).

---

*Part of the [ALTER DBA ADD AGENT](/ai-for-dbas/alter-dba-add-agent/) series.*
