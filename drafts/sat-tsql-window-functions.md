Window functions are one of those T-SQL features I use constantly but still have to pause and think about every time I write one. The `PARTITION BY` is straightforward. The `ORDER BY` makes sense. Then you need a frame clause — `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` — and suddenly you're checking Microsoft Learn for the third time this month.

This is exactly the kind of work where an AI agent shines. You know what result you want, and the agent knows the syntax cold. The combination is faster than either of you alone.

## Example 1: Row Numbering with Partitions

The most common window function request: "Give me the most recent order for each customer."

```text
Write a query against the OrderProcessing database that returns the most
recent order for each customer. Use ROW_NUMBER() partitioned by CustomerID,
ordered by OrderDate descending. Return CustomerID, CustomerName, OrderID,
OrderDate, and OrderTotal. Only return rows where the row number is 1.
Bracket all identifiers, use CONVERT not CAST, terminate with a semicolon.
```

The agent produces:

```sql
;WITH [RankedOrders] AS
(
    SELECT
        c.[CustomerID]
        , c.[CustomerName]
        , o.[OrderID]
        , o.[OrderDate]
        , o.[OrderTotal]
        , ROW_NUMBER() OVER (
            PARTITION BY c.[CustomerID]
            ORDER BY o.[OrderDate] DESC, o.[OrderID] DESC
        ) AS [RowNum]
    FROM [dbo].[Customers] AS c
    INNER JOIN [dbo].[Orders] AS o
        ON c.[CustomerID] = o.[CustomerID]
)
SELECT
    [CustomerID]
    , [CustomerName]
    , [OrderID]
    , [OrderDate]
    , [OrderTotal]
FROM [RankedOrders]
WHERE [RowNum] = 1;
```

Clean, correct, and follows the formatting conventions. If I needed `DENSE_RANK` instead (to handle ties on `OrderDate`), I can say so and the agent adjusts. One thing I appreciate: when you ask the agent to explain the difference between `ROW_NUMBER`, `RANK`, and `DENSE_RANK`, it gives a clear, concise answer with examples. That's useful when mentoring junior DBAs.

## Example 2: Running Totals

Running totals are where frame clauses become essential — and where the syntax starts to feel hostile.

```text
Write a query that calculates a running total of OrderTotal by customer,
ordered by OrderDate. Include CustomerID, OrderDate, OrderTotal, and the
running total column. The running total should accumulate from the first
order to the current row, per customer. Use the OrderProcessing database.
```

```sql
SELECT
    o.[CustomerID]
    , o.[OrderDate]
    , o.[OrderTotal]
    , SUM(o.[OrderTotal]) OVER (
        PARTITION BY o.[CustomerID]
        ORDER BY o.[OrderDate], o.[OrderID]
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS [RunningTotal]
FROM [dbo].[Orders] AS o
ORDER BY o.[CustomerID], o.[OrderDate], o.[OrderID];
```

The frame clause — `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` — is the part I always look up. The agent gets it right every time, and more importantly, it knows when the default frame is sufficient and when you need to be explicit.

A note on `ROWS` vs. `RANGE`: the agent correctly used `ROWS` here. `RANGE` would produce different results when there are duplicate `OrderDate` values for the same customer — `RANGE` treats ties as part of the same group, which can produce unexpected running totals. I always verify this choice matches my intent. If you're not sure about the difference, ask the agent to explain it — the explanation it gives is one of the clearest I've seen.

## Example 3: LAG/LEAD for Change Detection

This is the more complex case — comparing each row to the previous or next row within a partition.

```text
Write a query that shows each customer's orders with the previous order date
and the number of days between orders. Use LAG to get the previous order date
within each customer partition. Include CustomerID, OrderDate, OrderTotal,
PreviousOrderDate, and DaysSincePreviousOrder. Handle the first order per
customer (where there is no previous order) by showing NULL for both
PreviousOrderDate and DaysSincePreviousOrder. Use CONVERT for date arithmetic,
not DATEDIFF.
```

Here's where I caught something. The agent initially produced:

```sql
SELECT
    o.[CustomerID]
    , o.[OrderDate]
    , o.[OrderTotal]
    , LAG(o.[OrderDate]) OVER (
        PARTITION BY o.[CustomerID]
        ORDER BY o.[OrderDate]
    ) AS [PreviousOrderDate]
    , DATEDIFF(DAY,
        LAG(o.[OrderDate]) OVER (
            PARTITION BY o.[CustomerID]
            ORDER BY o.[OrderDate]
        ),
        o.[OrderDate]
    ) AS [DaysSincePreviousOrder]
FROM [dbo].[Orders] AS o
ORDER BY o.[CustomerID], o.[OrderDate];
```

Two things I changed:

1. **It used `DATEDIFF` despite the prompt saying `CONVERT`.** This is actually a case where the agent was right and my prompt was imprecise. `DATEDIFF` is the correct function for calculating the difference between two dates — `CONVERT` doesn't do date arithmetic. The prompt should have said "use `DATEDIFF` for the day calculation." Know when the agent is correctly pushing back on your instructions.

2. **The duplicate `LAG()` call.** The agent computed `LAG(o.[OrderDate])` twice — once for the column and once inside `DATEDIFF`. That's inefficient. I asked it to refactor using a CTE:

```sql
;WITH [OrdersWithPrevious] AS
(
    SELECT
        o.[CustomerID]
        , o.[OrderDate]
        , o.[OrderTotal]
        , LAG(o.[OrderDate]) OVER (
            PARTITION BY o.[CustomerID]
            ORDER BY o.[OrderDate]
        ) AS [PreviousOrderDate]
    FROM [dbo].[Orders] AS o
)
SELECT
    [CustomerID]
    , [OrderDate]
    , [OrderTotal]
    , [PreviousOrderDate]
    , DATEDIFF(DAY, [PreviousOrderDate], [OrderDate]) AS [DaysSincePreviousOrder]
FROM [OrdersWithPrevious]
ORDER BY [CustomerID], [OrderDate];
```

Now `LAG` is computed once, and the `DATEDIFF` references the CTE column. Cleaner and the optimizer only evaluates the window function once.

## Why the Agent Is Good at This

Window functions have rigid syntax rules — the `OVER` clause, partitioning, ordering, frame specification — and the agent has internalized all of them. Where humans trip up on frame clause syntax or forget whether `RANGE` and `ROWS` differ, the agent produces correct syntax on the first attempt in the vast majority of cases.

More importantly, the agent is excellent at *explaining* window functions. If you're mentoring someone, try this:

```text
Explain how this window function works, step by step:

SUM(o.[OrderTotal]) OVER (
    PARTITION BY o.[CustomerID]
    ORDER BY o.[OrderDate]
    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
)
```

The explanation it gives — breaking down what `PARTITION BY` does, how `ORDER BY` establishes the row sequence within each partition, and how the frame clause defines the "window" of rows included in the calculation — is clear enough to hand directly to a junior DBA.

## Try This Yourself

Start with a simple `ROW_NUMBER` query — the "most recent X per group" pattern. Then try:

1. A running total with `SUM() OVER (... ROWS BETWEEN ...)`.
2. A `LAG`/`LEAD` query to compare consecutive rows.
3. A `DENSE_RANK` query to identify ties.

For each one, ask the agent to write it, then ask it to explain each part of the `OVER` clause. Compare the explanation to what Microsoft Learn says. You'll find the agent's explanation is often more concise and practical.

For the broader workflow of writing and refining T-SQL with AI assistance, see [Writing T-SQL with an AI Partner](/ai-for-dbas/writing-tsql-with-ai-partner/).

---

*Part of the [ALTER DBA ADD AGENT](/ai-for-dbas/alter-dba-add-agent/) series.*
