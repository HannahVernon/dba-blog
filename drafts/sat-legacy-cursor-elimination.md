The [companion satellite post](/ai-for-dbas/ai-cursor-to-set-based-conversion/) covers straightforward cursor-to-set-based conversions — the cases where a `CASE` expression or a single `UPDATE...FROM` replaces the cursor cleanly. This post covers the harder cases: the legacy production cursors that have been running for years, are genuinely complex, and can't be solved with a single set-based statement.

These are the cursors with nested loops, conditional branching that calls other procedures mid-iteration, and side effects like audit logging, email notifications, or external API calls that must happen per-row. They're the ones where a junior developer looks at it and says "just rewrite it as a set-based query" — and a senior DBA says "it's not that simple."

AI agents are genuinely useful here, but the value isn't a magic one-click conversion. It's the decomposition: breaking a monolithic cursor into components, identifying which parts can go set-based and which parts must stay row-by-row.

## The Problem: A Legacy Order Fulfillment Procedure

Here's a simplified version of a pattern I've seen in multiple production systems. An order fulfillment procedure that processes each order individually with multiple conditional paths:

```sql
CREATE OR ALTER PROCEDURE [dbo].[usp_FulfillOrders]
    @BatchDate date
AS
BEGIN
    SET NOCOUNT ON;

    DECLARE @OrderID int, @CustomerID int, @OrderTotal decimal(18, 2);
    DECLARE @ShipMethod varchar(20), @WarehouseID int, @CustomerEmail varchar(200);
    DECLARE @TrackingNumber varchar(50), @FulfillmentStatus varchar(20);

    DECLARE [fulfillment_cursor] CURSOR LOCAL FAST_FORWARD FOR
        SELECT o.[OrderID], o.[CustomerID], o.[OrderTotal]
            , o.[ShipMethod], o.[WarehouseID], c.[Email]
        FROM [dbo].[Orders] AS o
        INNER JOIN [dbo].[Customers] AS c
            ON o.[CustomerID] = c.[CustomerID]
        WHERE o.[OrderDate] = @BatchDate
            AND o.[Status] = 'Approved';

    OPEN [fulfillment_cursor];
    FETCH NEXT FROM [fulfillment_cursor]
        INTO @OrderID, @CustomerID, @OrderTotal
            , @ShipMethod, @WarehouseID, @CustomerEmail;

    WHILE @@FETCH_STATUS = 0
    BEGIN
        SET @FulfillmentStatus = 'Pending';

        /* Check inventory availability */
        IF EXISTS (
            SELECT 1 FROM [dbo].[OrderItems] AS oi
            INNER JOIN [dbo].[Inventory] AS i
                ON oi.[ProductID] = i.[ProductID]
                AND i.[WarehouseID] = @WarehouseID
            WHERE oi.[OrderID] = @OrderID
                AND i.[QuantityOnHand] < oi.[Quantity]
        )
        BEGIN
            SET @FulfillmentStatus = 'BackOrdered';

            /* Log the backorder event */
            INSERT INTO [dbo].[FulfillmentLog] ([OrderID], [EventType], [EventDate], [Details])
            VALUES (@OrderID, 'BACKORDER', GETDATE(), 'Insufficient inventory at warehouse');

            /* Notify purchasing */
            EXEC [dbo].[usp_NotifyPurchasing] @OrderID = @OrderID
                , @WarehouseID = @WarehouseID;
        END
        ELSE
        BEGIN
            /* Reserve inventory */
            UPDATE i
            SET i.[QuantityOnHand] = i.[QuantityOnHand] - oi.[Quantity]
                , i.[QuantityReserved] = i.[QuantityReserved] + oi.[Quantity]
                , i.[LastUpdated] = GETDATE()
            FROM [dbo].[Inventory] AS i
            INNER JOIN [dbo].[OrderItems] AS oi
                ON i.[ProductID] = oi.[ProductID]
                AND i.[WarehouseID] = @WarehouseID
            WHERE oi.[OrderID] = @OrderID;

            /* Generate tracking number based on shipping method */
            IF @ShipMethod = 'Express'
                EXEC [dbo].[usp_GenerateTracking] @ShipMethod = 'Express'
                    , @OrderID = @OrderID
                    , @TrackingNumber = @TrackingNumber OUTPUT;
            ELSE
                EXEC [dbo].[usp_GenerateTracking] @ShipMethod = 'Standard'
                    , @OrderID = @OrderID
                    , @TrackingNumber = @TrackingNumber OUTPUT;

            SET @FulfillmentStatus = 'Fulfilled';

            /* Log the fulfillment event */
            INSERT INTO [dbo].[FulfillmentLog] ([OrderID], [EventType], [EventDate], [Details])
            VALUES (@OrderID, 'FULFILLED', GETDATE()
                , 'Tracking: ' + ISNULL(@TrackingNumber, 'N/A'));

            /* Send confirmation email */
            EXEC [msdb].[dbo].[sp_send_dbmail]
                @profile_name = 'OrderNotifications'
                , @recipients = @CustomerEmail
                , @subject = 'Your order has shipped'
                , @body = 'Your tracking number is: '
                    + ISNULL(@TrackingNumber, 'pending');
        END;

        /* Update order status regardless of path */
        UPDATE [dbo].[Orders]
        SET [Status] = @FulfillmentStatus
            , [FulfillmentDate] = GETDATE()
            , [TrackingNumber] = @TrackingNumber
        WHERE [OrderID] = @OrderID;

        /* Audit trail */
        INSERT INTO [dbo].[OrderAudit] ([OrderID], [Action], [ActionDate]
            , [ActionBy], [OldStatus], [NewStatus])
        VALUES (@OrderID, 'FULFILLMENT', GETDATE(), SYSTEM_USER
            , 'Approved', @FulfillmentStatus);

        FETCH NEXT FROM [fulfillment_cursor]
            INTO @OrderID, @CustomerID, @OrderTotal
                , @ShipMethod, @WarehouseID, @CustomerEmail;
    END;

    CLOSE [fulfillment_cursor];
    DEALLOCATE [fulfillment_cursor];
END;
```

This cursor does at least six distinct things per row: checks inventory, reserves stock, generates tracking numbers, logs events, sends emails, and writes audit records. You can't replace this with a single `UPDATE...FROM`.

## The Prompt

```text
Analyze this order fulfillment stored procedure. It uses a cursor to process
each order individually. I want to convert as much as possible to set-based
operations, but I know some parts (email, procedure calls with OUTPUT
parameters) must stay row-by-row.

Decompose this into:
1. Operations that CAN be done set-based (bulk updates, bulk inserts)
2. Operations that MUST remain row-by-row (procedure calls, email, anything
   with per-row side effects)
3. A rewritten version that does the set-based work first in bulk, then loops
   only for the parts that require it.

Bracket all identifiers, use semicolons, use CONVERT not CAST.
```

## What the Agent Produced

The agent broke the procedure into three phases, which is exactly the right decomposition.

**Phase 1 — Set-Based Classification:** Determine which orders can be fulfilled and which are backordered, in a single query, storing results in a temp table:

```sql
/* Phase 1: Classify all orders in a single pass */
SELECT
    o.[OrderID], o.[CustomerID], o.[OrderTotal]
    , o.[ShipMethod], o.[WarehouseID], c.[Email]
    , CASE
        WHEN EXISTS (
            SELECT 1 FROM [dbo].[OrderItems] AS oi
            INNER JOIN [dbo].[Inventory] AS i
                ON oi.[ProductID] = i.[ProductID]
                AND i.[WarehouseID] = o.[WarehouseID]
            WHERE oi.[OrderID] = o.[OrderID]
                AND i.[QuantityOnHand] < oi.[Quantity]
        ) THEN 'BackOrdered'
        ELSE 'Fulfilled'
      END AS [FulfillmentStatus]
INTO [#OrderBatch]
FROM [dbo].[Orders] AS o
INNER JOIN [dbo].[Customers] AS c
    ON o.[CustomerID] = c.[CustomerID]
WHERE o.[OrderDate] = @BatchDate
    AND o.[Status] = 'Approved';
```

**Phase 2 — Bulk Set-Based Operations:** All the inserts and updates that don't require per-row procedure calls:

```sql
/* Phase 2a: Bulk reserve inventory for all fulfillable orders */
UPDATE i
SET i.[QuantityOnHand] = i.[QuantityOnHand] - oi.[Quantity]
    , i.[QuantityReserved] = i.[QuantityReserved] + oi.[Quantity]
    , i.[LastUpdated] = GETDATE()
FROM [dbo].[Inventory] AS i
INNER JOIN [dbo].[OrderItems] AS oi
    ON i.[ProductID] = oi.[ProductID]
INNER JOIN [#OrderBatch] AS ob
    ON oi.[OrderID] = ob.[OrderID]
    AND i.[WarehouseID] = ob.[WarehouseID]
WHERE ob.[FulfillmentStatus] = 'Fulfilled';

/* Phase 2b: Bulk insert backorder log entries */
INSERT INTO [dbo].[FulfillmentLog] ([OrderID], [EventType], [EventDate], [Details])
SELECT ob.[OrderID], 'BACKORDER', GETDATE(), 'Insufficient inventory at warehouse'
FROM [#OrderBatch] AS ob
WHERE ob.[FulfillmentStatus] = 'BackOrdered';

/* Phase 2c: Bulk insert audit records */
INSERT INTO [dbo].[OrderAudit] ([OrderID], [Action], [ActionDate]
    , [ActionBy], [OldStatus], [NewStatus])
SELECT ob.[OrderID], 'FULFILLMENT', GETDATE(), SYSTEM_USER
    , 'Approved', ob.[FulfillmentStatus]
FROM [#OrderBatch] AS ob;

/* Phase 2d: Bulk update order status for backordered orders (no tracking needed) */
UPDATE o
SET o.[Status] = 'BackOrdered'
    , o.[FulfillmentDate] = GETDATE()
FROM [dbo].[Orders] AS o
INNER JOIN [#OrderBatch] AS ob
    ON o.[OrderID] = ob.[OrderID]
WHERE ob.[FulfillmentStatus] = 'BackOrdered';
```

**Phase 3 — Row-by-Row for Side Effects:** A reduced cursor that only loops through fulfilled orders and only handles the operations that require per-row processing:

```sql
/* Phase 3: Loop only for operations that require per-row execution */
DECLARE @OrderID int, @ShipMethod varchar(20), @CustomerEmail varchar(200);
DECLARE @TrackingNumber varchar(50);

DECLARE [fulfillment_cursor] CURSOR LOCAL FAST_FORWARD FOR
    SELECT ob.[OrderID], ob.[ShipMethod], ob.[Email]
    FROM [#OrderBatch] AS ob
    WHERE ob.[FulfillmentStatus] = 'Fulfilled';

OPEN [fulfillment_cursor];
FETCH NEXT FROM [fulfillment_cursor]
    INTO @OrderID, @ShipMethod, @CustomerEmail;

WHILE @@FETCH_STATUS = 0
BEGIN
    EXEC [dbo].[usp_GenerateTracking]
        @ShipMethod = @ShipMethod
        , @OrderID = @OrderID
        , @TrackingNumber = @TrackingNumber OUTPUT;

    /* Update order with tracking number */
    UPDATE [dbo].[Orders]
    SET [Status] = 'Fulfilled'
        , [FulfillmentDate] = GETDATE()
        , [TrackingNumber] = @TrackingNumber
    WHERE [OrderID] = @OrderID;

    /* Log fulfillment with tracking info */
    INSERT INTO [dbo].[FulfillmentLog] ([OrderID], [EventType], [EventDate], [Details])
    VALUES (@OrderID, 'FULFILLED', GETDATE()
        , 'Tracking: ' + ISNULL(@TrackingNumber, 'N/A'));

    EXEC [msdb].[dbo].[sp_send_dbmail]
        @profile_name = 'OrderNotifications'
        , @recipients = @CustomerEmail
        , @subject = 'Your order has shipped'
        , @body = 'Your tracking number is: '
            + ISNULL(@TrackingNumber, 'pending');

    FETCH NEXT FROM [fulfillment_cursor]
        INTO @OrderID, @ShipMethod, @CustomerEmail;
END;

CLOSE [fulfillment_cursor];
DEALLOCATE [fulfillment_cursor];

DROP TABLE [#OrderBatch];
```

## What I Validated and Changed

The decomposition was correct, but I caught several issues:

1. **Inventory race condition.** The original cursor checked inventory and reserved it row by row. The rewrite checks inventory in Phase 1 but reserves it in Phase 2. Between those two operations, another concurrent batch could consume the same inventory. I wrapped Phases 1 and 2 in an explicit transaction with `SERIALIZABLE` isolation on the inventory reads — or better, used `UPDLOCK, HOLDLOCK` hints on the inventory check in Phase 1. The agent didn't flag this because it wasn't thinking about concurrency.

2. **The `usp_NotifyPurchasing` call for backorders was dropped.** The agent moved the backorder logging to a bulk insert (correct) but forgot the procedure call that notifies purchasing. I added a second cursor for backordered items that calls `usp_NotifyPurchasing` per row — or, if that procedure just sends email, consolidated it into a single notification with all backordered order IDs.

3. **`GETDATE()` consistency.** In the original cursor, each row gets its own timestamp. In the set-based phases, all rows get the same `GETDATE()` value. I captured `GETDATE()` into a `@BatchTimestamp` variable at the top of the procedure and used it throughout for consistency.

4. **Email volume.** The cursor sends one email per fulfilled order. For a batch of 5,000 orders, that's 5,000 individual `sp_send_dbmail` calls. I flagged this as a separate optimization opportunity — batch the emails into a summary notification, or queue them through a Service Broker queue instead of blocking the fulfillment process. The agent's rewrite preserved the original behavior (one email per order), which is correct as a first pass, but it's worth questioning whether the original behavior is *desirable*.

## When Cursors Can't Be Eliminated

Let me be direct about this. Some row-by-row operations genuinely can't go set-based:

- **`sp_send_dbmail`** — one call per message, no set-based alternative
- **Stored procedure calls with `OUTPUT` parameters** — the tracking number generation here *must* be called per row because each call returns a different value
- **External API calls via CLR or `xp_cmdshell`** — inherently sequential
- **Operations where row N depends on row N-1** — running balance calculations where each row's result feeds the next

The agent recognized all of these correctly. The goal isn't zero cursors — it's *minimal* cursors. Move everything you can to set-based operations, then loop only for the operations that require it. The Phase 1/2/3 pattern the agent produced is the right architectural approach.

In the example above, the original cursor executed roughly 7 operations per row (inventory check, reserve, generate tracking, two log inserts, status update, email). The rewritten version executes 3 operations per row (generate tracking, status update, email) and handles the rest in bulk. For a batch of 1,000 orders, that's 4,000 fewer individual SQL statements — with the heaviest operations (inventory updates, audit inserts) now running as efficient set-based bulk operations.

## Try This Yourself

Find a complex cursor in your environment — the kind with multiple `IF` branches and side effects. Feed it to the agent with this prompt:

```text
Decompose this cursor into set-based and row-by-row components. Identify
which operations can be done as bulk INSERT/UPDATE/DELETE statements and
which must remain in a loop due to per-row side effects (procedure calls,
email, OUTPUT parameters). Rewrite it using the bulk-first, loop-for-
side-effects pattern.
```

Compare the rewrite's behavior against the original on a test system. Pay close attention to concurrency — the set-based version may have different locking behavior than the row-by-row version, and that difference can introduce blocking or deadlocks under load.

For the simpler cursor-to-set-based conversions, see the [companion post on cursor elimination](/ai-for-dbas/ai-cursor-to-set-based-conversion/). For the full legacy code reverse-engineering workflow, see [Reverse-Engineering Legacy Stored Procedures](/ai-for-dbas/reverse-engineering-legacy-procedures/). For T-SQL generation and refactoring patterns, see [Writing T-SQL with an AI Partner](/ai-for-dbas/writing-tsql-with-ai-partner/).

---

*Part of the [ALTER DBA ADD AGENT](/ai-for-dbas/alter-dba-add-agent/) series.*
