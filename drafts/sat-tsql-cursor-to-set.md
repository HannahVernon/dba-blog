Few things make a DBA's eye twitch faster than opening a stored procedure and seeing `DECLARE CURSOR`. Cursors process rows one at a time in an engine designed to process sets. They're almost always slower, harder to read, and more fragile than the set-based alternative. But they keep showing up in production — usually written by application developers who think in loops, or inherited from someone who left the company years ago.

The good news: converting cursor-based logic to set-based operations is one of the things AI agents do exceptionally well. The agent can see the row-by-row pattern, understand the intent, and produce a set-based equivalent in seconds. Your job is verifying it's correct.

## The Problem: A Row-by-Row Status Update

Here's a realistic cursor — the kind you find in order processing systems everywhere. It iterates through pending orders, applies business rules, and updates their status one row at a time:

```sql
CREATE OR ALTER PROCEDURE [dbo].[usp_ProcessPendingOrders]
AS
BEGIN
    SET NOCOUNT ON;

    DECLARE @OrderID int;
    DECLARE @OrderTotal decimal(18, 2);
    DECLARE @CustomerTier varchar(20);
    DECLARE @NewStatus varchar(20);

    DECLARE [OrderCursor] CURSOR LOCAL FAST_FORWARD FOR
        SELECT o.[OrderID], o.[OrderTotal], c.[CustomerTier]
        FROM [dbo].[Orders] AS o
        INNER JOIN [dbo].[Customers] AS c
            ON o.[CustomerID] = c.[CustomerID]
        WHERE o.[Status] = 'Pending'
            AND o.[OrderDate] < DATEADD(DAY, -1, GETDATE());

    OPEN [OrderCursor];
    FETCH NEXT FROM [OrderCursor] INTO @OrderID, @OrderTotal, @CustomerTier;

    WHILE @@FETCH_STATUS = 0
    BEGIN
        /* Determine new status based on business rules */
        IF @OrderTotal > 5000.00 AND @CustomerTier = 'Premium'
            SET @NewStatus = 'AutoApproved';
        ELSE IF @OrderTotal > 10000.00
            SET @NewStatus = 'RequiresReview';
        ELSE IF @OrderTotal <= 0.00
            SET @NewStatus = 'Cancelled';
        ELSE
            SET @NewStatus = 'Approved';

        UPDATE [dbo].[Orders]
        SET [Status] = @NewStatus
            , [ProcessedDate] = GETDATE()
            , [ProcessedBy] = 'SYSTEM'
        WHERE [OrderID] = @OrderID;

        FETCH NEXT FROM [OrderCursor] INTO @OrderID, @OrderTotal, @CustomerTier;
    END;

    CLOSE [OrderCursor];
    DEALLOCATE [OrderCursor];
END;
```

This works. It's even using `FAST_FORWARD` — which is the least-bad cursor option. But if you have 50,000 pending orders, you're issuing 50,000 individual `UPDATE` statements. Each one acquires locks, writes to the transaction log, and releases locks. Individually. That's a lot of unnecessary overhead.

## The Prompt

```text
Rewrite this cursor-based stored procedure as a single set-based UPDATE
statement. Preserve the exact same business logic — the same conditions
should produce the same status values. Use a CASE expression for the
status determination. Bracket all identifiers, use semicolons, and add
a brief comment explaining the business rules.
```

## What the Agent Produced

```sql
CREATE OR ALTER PROCEDURE [dbo].[usp_ProcessPendingOrders]
AS
BEGIN
    SET NOCOUNT ON;

    /* Process all pending orders older than 24 hours in a single pass.
       Status rules:
         - Premium customers with orders > $5,000 → AutoApproved
         - Any order > $10,000 → RequiresReview
         - Zero or negative totals → Cancelled
         - Everything else → Approved
       Rule priority: evaluated top-to-bottom via CASE expression. */
    UPDATE o
    SET [Status] = CASE
            WHEN o.[OrderTotal] > 5000.00 AND c.[CustomerTier] = 'Premium'
                THEN 'AutoApproved'
            WHEN o.[OrderTotal] > 10000.00
                THEN 'RequiresReview'
            WHEN o.[OrderTotal] <= 0.00
                THEN 'Cancelled'
            ELSE 'Approved'
        END
        , [ProcessedDate] = GETDATE()
        , [ProcessedBy] = 'SYSTEM'
    FROM [dbo].[Orders] AS o
    INNER JOIN [dbo].[Customers] AS c
        ON o.[CustomerID] = c.[CustomerID]
    WHERE o.[Status] = 'Pending'
        AND o.[OrderDate] < DATEADD(DAY, -1, GETDATE());
END;
```

One statement. Same result. The `CASE` expression evaluates the same conditions in the same order as the cursor's `IF`/`ELSE IF` chain. The `WHERE` clause matches the cursor's `SELECT` filter exactly.

## What I Validated

The conversion looks clean, but I check three things:

1. **CASE evaluation order.** SQL Server evaluates `CASE` branches top-to-bottom and stops at the first match — same as the `IF`/`ELSE IF` chain. This matters here because a Premium customer with a $12,000 order matches both the first and second conditions. In the cursor, the `IF` chain catches it at the first condition (AutoApproved). In the `CASE` expression, same thing — first matching `WHEN` wins. Semantics preserved.

2. **Row count comparison.** I run both versions against a test dataset and compare the number of rows affected and the resulting status distribution. They should be identical.

3. **GETDATE() consistency.** In the cursor, `GETDATE()` is called once per row — meaning each order gets a slightly different `ProcessedDate`. In the set-based version, `GETDATE()` is evaluated once for the entire statement — all orders get the same timestamp. This is usually *better* behavior (consistent batch timestamp), but if the original code intentionally relied on per-row timestamps, it's a semantic change worth noting.

## The Performance Difference

On a test set of 25,000 pending orders:

| Approach | Duration | Logical Reads |
|---|---|---|
| Cursor (FAST_FORWARD) | ~14 seconds | ~425,000 |
| Set-based UPDATE | ~0.4 seconds | ~52,000 |

The set-based version is roughly 35x faster. The logical reads drop by an order of magnitude because the engine reads and updates the pages in bulk rather than seeking to each row individually. Transaction log writes are also consolidated — one large write instead of thousands of small ones.

## When Cursors Are Still Appropriate

I want to be fair here. Cursors aren't always wrong. There are legitimate cases:

- **Sending database mail row by row.** You can't send emails in a set-based operation. Each row needs its own `sp_send_dbmail` call.
- **Dynamic SQL per row.** If each row requires a different dynamic SQL statement (e.g., processing different tables based on metadata), a cursor or `WHILE` loop is appropriate.
- **Small rowsets with complex per-row logic.** If you're processing 50 rows and each row triggers an API call or external system interaction, the cursor overhead is irrelevant.
- **Ordered processing with dependencies.** When the processing of row N depends on the result of processing row N-1, set-based operations may not work. Running totals can sometimes substitute, but not always.

The agent knows these cases too. If you feed it a cursor that legitimately can't be set-based, it will usually tell you — or suggest a hybrid approach where the set-based portion handles the data retrieval and the loop handles the per-row operation that requires sequential processing.

## Try This Yourself

Find a cursor in your environment — nearly every SQL Server instance has at least one. Feed it to the agent with this prompt:

```text
Can this cursor be rewritten as a set-based operation? If yes, rewrite it.
If parts of it must remain row-by-row, explain which parts and why.
Preserve exact business logic semantics.
```

Compare the results on a test system. Check row counts, status distributions, and edge cases. If the set-based version produces identical results and runs faster, you've just improved your codebase.

For the full T-SQL workflow including code generation, review, and refactoring, see [Writing T-SQL with an AI Partner](/ai-for-dbas/writing-tsql-with-ai-partner/). For tackling entire legacy procedures end-to-end, see [Reverse Engineering Legacy Stored Procedures](/ai-for-dbas/reverse-engineering-legacy-procedures/).

---

*Part of the [ALTER DBA ADD AGENT](/ai-for-dbas/alter-dba-add-agent/) series.*
