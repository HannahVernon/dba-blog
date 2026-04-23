Every SQL Server environment has them — stored procedures written in the SQL Server 2000 era where every join is an implicit comma join in the `FROM` clause, with join conditions buried deep in the `WHERE` clause alongside filter predicates. They work. They've been working for twenty years. But they're a maintenance hazard, and sooner or later you have to modernize them.

The real risk isn't readability (though that's bad enough). It's accidental cross joins. Remove or misplace a single `WHERE` condition during a maintenance change and you silently get a cartesian product. With explicit ANSI `JOIN` syntax, a missing `ON` clause is a syntax error — the engine won't let you run it. That safety net matters.

I've used AI agents extensively for this conversion work, and it's one of the tasks they handle remarkably well. Here's my workflow.

## The Problem: Legacy Implicit Joins

Here's a realistic example — a reporting procedure pulling order data across five tables, all joined with comma syntax:

```sql
CREATE OR ALTER PROCEDURE [dbo].[usp_OrderSummaryReport]
    @StartDate datetime,
    @EndDate datetime
AS
BEGIN
    SET NOCOUNT ON;

    SELECT
        c.[CustomerName]
        , c.[AccountNumber]
        , o.[OrderID]
        , o.[OrderDate]
        , p.[ProductName]
        , p.[SKU]
        , od.[Quantity]
        , od.[UnitPrice]
        , od.[Quantity] * od.[UnitPrice] AS [LineTotal]
        , s.[ShipperName]
        , o.[ShippedDate]
    FROM
        [dbo].[Customers] c
        , [dbo].[Orders] o
        , [dbo].[OrderDetails] od
        , [dbo].[Products] p
        , [dbo].[Shippers] s
    WHERE c.[CustomerID] = o.[CustomerID]
        AND o.[OrderID] = od.[OrderID]
        AND od.[ProductID] = p.[ProductID]
        AND o.[ShipperID] = s.[ShipperID]
        AND o.[OrderDate] >= @StartDate
        AND o.[OrderDate] < @EndDate
        AND c.[IsActive] = 1;
END;
```

Five tables, five lines in the `WHERE` clause doing double duty as join conditions and filter predicates. Now imagine this is 400 lines long with subqueries, `CASE` expressions, and comments that haven't been accurate since 2011. That's where the agent earns its keep.

## The Prompt

```text
Convert this stored procedure from implicit (comma) joins to explicit ANSI
JOIN syntax. Move all join conditions to ON clauses and leave only filter
predicates in the WHERE clause. Preserve the exact same query semantics —
same result set, same column order, same filtering behavior. Use INNER JOIN
unless the original logic requires an outer join. Bracket all identifiers,
terminate statements with semicolons, and use /* */ for any comments.
```

I paste the procedure after the prompt, or reference the file if the agent has access to it.

## What the Agent Produced

```sql
CREATE OR ALTER PROCEDURE [dbo].[usp_OrderSummaryReport]
    @StartDate datetime,
    @EndDate datetime
AS
BEGIN
    SET NOCOUNT ON;

    SELECT
        c.[CustomerName]
        , c.[AccountNumber]
        , o.[OrderID]
        , o.[OrderDate]
        , p.[ProductName]
        , p.[SKU]
        , od.[Quantity]
        , od.[UnitPrice]
        , od.[Quantity] * od.[UnitPrice] AS [LineTotal]
        , s.[ShipperName]
        , o.[ShippedDate]
    FROM [dbo].[Customers] AS c
    INNER JOIN [dbo].[Orders] AS o
        ON c.[CustomerID] = o.[CustomerID]
    INNER JOIN [dbo].[OrderDetails] AS od
        ON o.[OrderID] = od.[OrderID]
    INNER JOIN [dbo].[Products] AS p
        ON od.[ProductID] = p.[ProductID]
    INNER JOIN [dbo].[Shippers] AS s
        ON o.[ShipperID] = s.[ShipperID]
    WHERE o.[OrderDate] >= @StartDate
        AND o.[OrderDate] < @EndDate
        AND c.[IsActive] = 1;
END;
```

The agent correctly separated join conditions from filter predicates. The `CustomerID`, `OrderID`, `ProductID`, and `ShipperID` equalities moved to `ON` clauses. The date range and `IsActive` filter stayed in `WHERE` — exactly where they belong.

## What I Validated

For inner joins, this conversion is semantically equivalent. The query optimizer treats implicit and explicit inner joins identically — same execution plan, same results. But I still verify:

1. **Row count comparison.** Run both versions against the same date range. Identical row counts are the first check.
2. **Execution plan diff.** Compare the actual execution plans. For inner joins, they should be identical. If they differ, investigate why.
3. **Edge case: predicates that belong in ON vs. WHERE.** With inner joins this doesn't matter semantically, but if someone later changes an `INNER JOIN` to a `LEFT JOIN`, the distinction becomes critical. A filter in `ON` filters before the join; a filter in `WHERE` filters after — producing very different results for outer joins.
4. **Edge case: accidental implicit cross joins.** If the original query had a table in the `FROM` list with no corresponding `WHERE` condition, that's a cross join — intentional or not. The agent should flag this. In my experience, agents consistently catch these and ask whether the cross join is intended.

The agent's output was correct for this example. No changes needed.

## Where It Gets Tricky

The straightforward cases — equi-joins between primary and foreign keys — are easy. The conversions that require judgment include:

- **Mixed inner and outer joins.** Legacy code sometimes uses `*=` or `=*` (the old proprietary outer join syntax, removed in SQL Server 2012). The agent needs to convert these to `LEFT JOIN` or `RIGHT JOIN` while preserving the semantics. I've found agents handle this well, but always verify the row counts.
- **Self-joins with multiple conditions.** When a table joins to itself with multiple `WHERE` conditions, the agent needs to determine which conditions are join conditions and which are filters. Context matters here.
- **Correlated subqueries in WHERE that are actually joins.** Sometimes legacy code uses `WHERE EXISTS` or `WHERE column IN (SELECT ...)` where a join would be clearer. The agent may or may not suggest this refactoring — you can prompt for it explicitly.

For a deeper look at tackling entire legacy stored procedures — not just join conversion but full-procedure reverse engineering and modernization — see [Reverse Engineering Legacy Stored Procedures](/ai-for-dbas/reverse-engineering-legacy-procedures/).

## Try This Yourself

Pick a stored procedure in your environment that uses implicit joins. Before you touch it:

1. Run it with a known set of parameters and capture the result set and row count.
2. Capture the actual execution plan.
3. Feed it to the agent with the prompt above.
4. Run the converted version with the same parameters and compare results, row counts, and plans.

Start with a simple procedure — three or four tables, all inner joins. Once you're comfortable with the workflow, try one with outer joins or self-joins. The agent handles these correctly more often than not, but "more often than not" means you need to verify every time.

The general workflow for writing and refactoring T-SQL with AI is covered in [Writing T-SQL with an AI Partner](/ai-for-dbas/writing-tsql-with-ai-partner/).

---

*Part of the [ALTER DBA ADD AGENT](/ai-for-dbas/alter-dba-add-agent/) series.*
