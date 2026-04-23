Deadlock graphs contain everything you need to diagnose the problem. The resource list tells you which objects are involved. The process list tells you what each session was doing. The victim list tells you who lost. It's all there — buried in XML that takes 10 minutes to parse manually at 2 AM when you're getting paged.

This is a good use case for an AI agent: structured data that requires pattern recognition and explanation. You're not asking the agent to *fix* anything. You're asking it to read the XML faster than you can and explain what happened in plain language.

## Collecting Deadlock Graphs

Before you can analyze a deadlock, you need to capture it. Three main approaches:

**The system_health session.** SQL Server captures deadlock graphs automatically in the `system_health` Extended Events session. This is always running — you don't need to configure anything. The catch: it only retains a limited number of events, so if you don't check regularly, older deadlocks roll off.

```sql
/* Extract deadlock graphs from system_health */
SELECT
    xdr.value(N'@timestamp', N'datetime2') AS [deadlock_time]
  , xdr.query(N'.') AS [deadlock_graph]
FROM
(
    SELECT CONVERT(xml, [target_data]) AS [target_data]
    FROM sys.dm_xe_session_targets AS st
    INNER JOIN sys.dm_xe_sessions AS s ON s.[address] = st.[event_session_address]
    WHERE s.[name] = N'system_health'
      AND st.[target_name] = N'ring_buffer'
) AS d
CROSS APPLY d.[target_data].nodes(N'RingBufferTarget/event[@name="xml_deadlock_report"]/data/value/deadlock') AS xd(xdr)
ORDER BY [deadlock_time] DESC;
```

**A dedicated Extended Events session.** For production systems where deadlocks are frequent, I set up a dedicated session that writes to a file target with longer retention. This gives you a complete history.

**Monitoring tools.** If you're running a third-party monitoring tool, it's likely already capturing deadlock graphs. Check there first.

## Feeding the Graph to the Agent

Here's a realistic deadlock scenario. Two stored procedures update the same two tables but in opposite order — the classic deadlock pattern.

```text
Analyze this deadlock XML graph and explain:
1. What resources are involved (tables, indexes, lock modes)?
2. What each process was doing at the time of the deadlock.
3. Which process was chosen as the victim and why.
4. What access pattern created this deadlock.
5. How to prevent it from recurring.

<deadlock>
  <victim-list>
    <victimProcess id="process2d8a1c8a8" />
  </victim-list>
  <process-list>
    <process id="process2d8a1c468" taskpriority="0" logused="4120"
             waitresource="KEY: 7:72057594045792256 (a1b2c3d4e5f6)"
             waittime="3847" ownerId="198234567" transactionname="user_transaction"
             lasttranstarted="2025-01-15T14:23:01.447" lockMode="X"
             schedulerid="4" kpid="8432" hostname="APP-WEB-03"
             loginname="svc_orderapp" isolationlevel="read committed (2)"
             currentdb="7" currentdbname="OrderProcessing">
      <executionStack>
        <frame procname="OrderProcessing.dbo.usp_ProcessReturn"
               line="18" sqlhandle="0x03000700a1b2c3d4...">
          UPDATE [dbo].[OrderDetails] SET [Status] = @NewStatus
          WHERE [OrderID] = @OrderID AND [LineItemID] = @LineItemID
        </frame>
      </executionStack>
      <inputbuf>EXEC dbo.usp_ProcessReturn @OrderID = 884523,
        @LineItemID = 3, @NewStatus = 'Returned'</inputbuf>
    </process>
    <process id="process2d8a1c8a8" taskpriority="0" logused="3856"
             waitresource="KEY: 7:72057594045464576 (f6e5d4c3b2a1)"
             waittime="2901" ownerId="198234590" transactionname="user_transaction"
             lasttranstarted="2025-01-15T14:23:01.892" lockMode="X"
             schedulerid="7" kpid="12044" hostname="APP-WEB-05"
             loginname="svc_orderapp" isolationlevel="read committed (2)"
             currentdb="7" currentdbname="OrderProcessing">
      <executionStack>
        <frame procname="OrderProcessing.dbo.usp_UpdateShipment"
               line="24" sqlhandle="0x03000700d4c3b2a1...">
          UPDATE [dbo].[Orders] SET [ShipDate] = @ShipDate,
          [TrackingNumber] = @TrackingNumber
          WHERE [OrderID] = @OrderID
        </frame>
      </executionStack>
      <inputbuf>EXEC dbo.usp_UpdateShipment @OrderID = 884523,
        @ShipDate = '2025-01-15', @TrackingNumber = 'TRK-99281'</inputbuf>
    </process>
  </process-list>
  <resource-list>
    <keylock hobtid="72057594045792256"
             dbid="7" objectname="OrderProcessing.dbo.OrderDetails"
             indexname="PK_OrderDetails" mode="X">
      <owner-list>
        <owner id="process2d8a1c8a8" mode="X" />
      </owner-list>
      <waiter-list>
        <waiter id="process2d8a1c468" mode="X" requestType="wait" />
      </waiter-list>
    </keylock>
    <keylock hobtid="72057594045464576"
             dbid="7" objectname="OrderProcessing.dbo.Orders"
             indexname="PK_Orders" mode="X">
      <owner-list>
        <owner id="process2d8a1c468" mode="X" />
      </owner-list>
      <waiter-list>
        <waiter id="process2d8a1c8a8" mode="X" requestType="wait" />
      </waiter-list>
    </keylock>
  </resource-list>
</deadlock>
```

## What the Agent Produces

The agent breaks down the deadlock clearly:

**Resources.** Two exclusive key locks — one on `PK_OrderDetails` (held by `usp_UpdateShipment`, wanted by `usp_ProcessReturn`) and one on `PK_Orders` (held by `usp_ProcessReturn`, wanted by `usp_UpdateShipment`). Both are exclusive (X) locks on clustered index keys.

**The cycle.** `usp_ProcessReturn` updated `dbo.Orders` first (acquiring an X lock), then tried to update `dbo.OrderDetails`. Meanwhile, `usp_UpdateShipment` updated `dbo.OrderDetails` first, then tried to update `dbo.Orders`. Classic opposite-order access pattern on the same `OrderID`.

**The victim.** Process `process2d8a1c8a8` (`usp_UpdateShipment`) was chosen as the victim because it had lower `logused` (3,856 bytes vs. 4,120 bytes) — SQL Server picks the least expensive transaction to roll back by default.

**The fix.** The agent suggests three approaches, ranked by preference:

1. **Consistent access order.** Refactor both procedures to always access `dbo.Orders` before `dbo.OrderDetails`. This eliminates the cycle entirely.
2. **Shorter transactions.** If the procedures are doing other work between the two updates, restructure so the conflicting updates happen as close together as possible.
3. **Read committed snapshot isolation (RCSI).** This wouldn't help here since both processes are taking X locks for writes, but the agent correctly notes this. I've seen agents incorrectly suggest RCSI for write-write deadlocks — always validate.

## What I Validated

The agent's analysis was accurate. The root cause was obvious once explained: two procedures, same tables, opposite order. But I checked two things the agent couldn't:

First, I looked at the full procedure definitions — not just the lines shown in the deadlock graph. `usp_ProcessReturn` had an explicit `BEGIN TRANSACTION` wrapping six statements when only two needed to be atomic. Narrowing the transaction scope reduced the lock hold time.

Second, I checked the deadlock frequency. This was happening 40–50 times per day during peak hours, which meant it wasn't a rare race condition — it was a systematic design problem that needed the access-order fix, not just retry logic.

## The Fixed Pattern

```sql
CREATE OR ALTER PROCEDURE dbo.[usp_ProcessReturn]
    @OrderID int
  , @LineItemID int
  , @NewStatus varchar(20)
AS
BEGIN
    SET NOCOUNT ON;

    /* Validation and lookups OUTSIDE the transaction */
    DECLARE @CurrentStatus varchar(20);
    SELECT @CurrentStatus = [Status]
    FROM dbo.[OrderDetails]
    WHERE [OrderID] = @OrderID
      AND [LineItemID] = @LineItemID;

    IF @CurrentStatus IS NULL
    BEGIN
        RAISERROR(N'Line item not found.', 16, 1);
        RETURN;
    END;

    /* Narrow transaction: consistent order (Orders first, then OrderDetails) */
    SET XACT_ABORT ON;

    BEGIN TRY
        BEGIN TRANSACTION;

            UPDATE dbo.[Orders]
            SET [LastModified] = SYSDATETIME()
            WHERE [OrderID] = @OrderID;

            UPDATE dbo.[OrderDetails]
            SET [Status] = @NewStatus
            WHERE [OrderID] = @OrderID
              AND [LineItemID] = @LineItemID;

        COMMIT TRANSACTION;
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0
            ROLLBACK TRANSACTION;

        ; THROW;
    END CATCH;
END;
```

The key change: both `usp_ProcessReturn` and `usp_UpdateShipment` now access `dbo.Orders` before `dbo.OrderDetails`. Same order, no cycle, no deadlock.

## Try This Yourself

1. Pull a recent deadlock graph from the `system_health` session using the query above.
2. Paste the full XML into your AI agent and ask for a human-readable analysis.
3. Verify the agent's findings against the actual procedure definitions and table schemas.
4. Check whether the fix the agent suggests is the *right* fix for your situation — consistent access order isn't always possible when third-party code is involved.
5. After applying the fix, monitor for deadlock recurrence using an Extended Events session with a file target.

The agent reads XML faster than you do. But the decision about *which* fix to apply — refactor procedures, add retry logic, change isolation levels — requires understanding your application's transaction semantics. That's your job.

For more on AI-assisted troubleshooting workflows, see [Post 8: Wait Stats, Deadlocks, and Blocking Chains](/ai-for-dbas/wait-stats-deadlocks-blocking/). For using AI during active incidents, see [Post 9: Incident Response and Root-Cause Analysis](/ai-for-dbas/incident-response-root-cause/).

---

*Part of the [ALTER DBA ADD AGENT](/ai-for-dbas/alter-dba-add-agent/) series.*
