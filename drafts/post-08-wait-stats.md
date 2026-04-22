# Wait Stats, Deadlocks, and Blocking Chains: AI-Assisted Diagnosis

It's 2 PM on a Tuesday. The application team says "the database is slow." You pull up wait stats and see `CXPACKET` through the roof. Or you get paged at midnight because the monitoring system flagged a deadlock. Or you open your inbox to find a blocking chain screenshot from a developer who doesn't know what they're looking at.

This is the troubleshooting workflow every DBA knows. The AI agent doesn't replace your diagnostic skills — but it can speed up the analysis phase when you're under pressure.

One important baseline note: wait stats should be captured over a *specific interval*, not read blindly since the last service restart. If you're not already doing differential captures, that's step zero.

## Wait Stats Analysis

Wait stats are the first thing most DBAs check when diagnosing performance problems. The data itself is easy to collect — the interpretation is where experience matters.

```text
Here are the top 20 wait types from our production instance captured over the
last hour. Analyze them and tell me:
1. Which waits indicate a real problem vs. normal background activity?
2. What's the likely root cause for the top problematic waits?
3. What should I investigate next for each one?
4. Any quick configuration changes that might help?

CXPACKET          847,293 ms    42.1%
SOS_SCHEDULER_YIELD  312,847 ms    15.5%
PAGEIOLATCH_SH    287,432 ms    14.3%
LCK_M_IX          198,744 ms     9.9%
ASYNC_NETWORK_IO   156,221 ms     7.8%
WRITELOG           89,432 ms     4.4%
...
```

The agent flags `CXPACKET` as parallel query overhead — though on modern SQL Server, `CXPACKET` alone isn't actionable; you need to check `CXCONSUMER` and correlate with actual query plans, DOP settings, and parallelism skew. It identifies `PAGEIOLATCH_SH` as physical page read waits that could indicate scan-heavy plans, missing indexes, memory pressure, or actual storage latency — recommend correlating with `sys.dm_io_virtual_file_stats` to distinguish. It notes that `ASYNC_NETWORK_IO` is usually a client-side consumption issue (row-by-row fetching from the app tier), not a SQL Server problem. And it calls out `LCK_M_IX` as lock contention worth investigating.

More importantly, it suggests *next steps*: check `sys.dm_exec_query_stats` for the highest-CPU queries driving the parallelism, review `MAXDOP` and `cost threshold for parallelism` settings, look at Query Store for plan regressions.

Is the agent always right? No. But it helps me work through the decision tree faster, especially when I'm fielding Teams messages at the same time.

## Deadlock XML Analysis

Deadlock graphs are powerful diagnostic data wrapped in an impenetrable XML format. Most DBAs can read them, but it takes time — and at 2 AM, time is the one thing you don't have.

```text
Analyze this deadlock XML and explain:
1. What resources are involved (tables, indexes, lock modes)?
2. What statements are the two processes running?
3. Which process was chosen as the victim and why?
4. What's the root cause — what access pattern created this deadlock?
5. How would you fix it?

<deadlock>
  <victim-list>
    <victimProcess id="process2" />
  </victim-list>
  <process-list>
    <process id="process1" waitresource="KEY: 7:72057594045202432 (a1b2c3d4e5f6)"
             waittime="3847" ownerId="892743621" transactionname="user_transaction"
             isolationlevel="read committed (2)" ...>
      <inputbuf>
        UPDATE dbo.OrderDetails SET Status = 'Shipped'
        WHERE OrderID = 48291 AND LineItemID = 3
      </inputbuf>
    </process>
    <process id="process2" waitresource="KEY: 7:72057594045267968 (f6e5d4c3b2a1)"
             waittime="2913" ownerId="892743658" transactionname="user_transaction"
             isolationlevel="read committed (2)" ...>
      <inputbuf>
        UPDATE dbo.Inventory SET QuantityReserved = QuantityReserved - 1
        WHERE ProductID = 7842
      </inputbuf>
    </process>
  </process-list>
  <resource-list>
    <keylock hobtid="72057594045202432" dbid="7" objectname="OrderDetails"
             indexname="PK_OrderDetails" mode="X">
      <owner-list><owner id="process1" mode="X" /></owner-list>
      <waiter-list><waiter id="process2" mode="X" requestType="wait" /></waiter-list>
    </keylock>
    <keylock hobtid="72057594045267968" dbid="7" objectname="Inventory"
             indexname="PK_Inventory" mode="X">
      <owner-list><owner id="process2" mode="X" /></owner-list>
      <waiter-list><waiter id="process1" mode="X" requestType="wait" /></waiter-list>
    </keylock>
  </resource-list>
</deadlock>
```

The agent parses the XML, identifies the resources, and explains the cycle: Process 1 holds an exclusive key lock on `OrderDetails` and wants a lock on `Inventory`; Process 2 holds an exclusive key lock on `Inventory` and wants a lock on `OrderDetails`. It identifies the access pattern and suggests fixes: ensure both code paths acquire locks in the same order, reduce the transaction scope, or consider whether a missing index is extending lock duration.

A caveat: the `inputbuf` only shows the *waiting* statement, not the full transaction history. The reason each process touched the second table might be a trigger, a foreign key enforcement, or an earlier statement not shown in the XML. The agent's explanation is a plausible hypothesis, not a certainty — you may need the full execution stack from Extended Events to confirm.

One thing the post would be incomplete without: **your application should have retry logic for error 1205** (deadlock victim). Deadlocks happen in any concurrent system. The question isn't whether they'll occur — it's whether your app handles them gracefully. If you don't have retry logic, that's a more important fix than the lock ordering.

Note: this is a simplified deadlock graph for illustration. Real-world deadlock XML is usually messier — proc names in the execution stack, parameter values, hostname/login context, and lock modes beyond the clean X/X pattern. The agent handles messy XML just as well; I've cleaned this one up so the cycle is easy to follow.

## Blocking Chain Triage

Long blocking chains during business hours are one of the most stressful DBA scenarios. You need to identify the head blocker, understand what it's doing, and decide whether to kill it — fast.

```text
Here's the output from sp_WhoIsActive. Analyze the blocking chain:
- Identify the head blocker
- What statement is it running? Is it actively executing or idle in an open transaction?
- How many sessions are blocked behind it, directly and indirectly?
- What's the total blocked time across all waiting sessions?
- What are my options — wait, kill, or something else?

session_id  status    blocking  open_tran  wait_info             sql_text                              host_name    program_name
55          sleeping  NULL      1          NULL                  UPDATE dbo.Accounts SET ...            APPSVR01     OrderService
72          suspended 55        0          LCK_M_S (00:03:12)    SELECT * FROM dbo.Accounts ...        RPTSVR01     SSRS
88          suspended 55        0          LCK_M_S (00:02:47)    SELECT * FROM dbo.Accounts ...        RPTSVR01     SSRS
91          suspended 72        0          LCK_M_IS (00:01:33)   SELECT a.Name FROM dbo.Accounts ...   APPSVR02     CustomerPortal
103         suspended 55        0          LCK_M_IX (00:01:12)   INSERT INTO dbo.AccountLog ...        APPSVR01     AuditWorker
```

The agent identifies session 55 as the head blocker — and critically, notes that it's *sleeping* with an open transaction, not actively executing. That's a common pattern: an application opened a transaction, did some work, and never committed — possibly waiting on an external service call or a user interaction. Session 55 holds an exclusive lock from its UPDATE, and everything else — the S locks from the report queries, the IX lock from the audit INSERT — is incompatible with that X lock and has to wait.

Before reaching for `KILL`, check the rollback cost. If session 55 has been running a large UPDATE, killing it means SQL Server has to roll back all that work — which can take longer than the original statement and keeps the locks held the entire time. Check `sys.dm_tran_active_transactions` for the transaction's log usage before deciding.

This is the kind of analysis I'd do myself — but having it done instantly means I can focus on the *decision* rather than the data gathering. One rule I follow: never let the agent make kill/no-kill decisions unreviewed. It can recommend, but the `KILL` command comes from me.

**Try This Yourself:** The next time you run `sp_WhoIsActive` during a blocking event, copy the output and paste it into the agent. Compare its analysis to your own.

## Generating Targeted Fixes

Once you've identified the problem, the agent helps you build the fix.

```text
The top wait type is PAGEIOLATCH_SH concentrated on the OrderHistory table.
The most frequent query is a report that scans the full clustered index
filtered by OrderDate. Write:
1. A covering index that might reduce or eliminate the scan
2. A query rewrite using a date range parameter instead of CONVERT on the column
3. A filtered index for the common case where Status = 'Active'
Describe the trade-offs for write performance with each option.
```

The agent generates all three options with qualitative trade-off analysis. It can't give you precise write-performance estimates without knowing your table size, DML rates, and existing index count — but it can tell you "a wide covering index on a write-heavy table will hurt more than a narrow filtered index." Your job is to evaluate which option fits your actual workload.

## The Agent as a Calm Second Opinion

During an incident, the most valuable thing the agent provides isn't speed — it's a checklist under pressure. It helps me avoid tunnel vision on the first hypothesis. It summarizes evidence while I'm handling the incident channel. It doesn't skip steps because someone is asking "is the database down?" for the third time.

One tip that's easy to overlook: you can paste screenshots directly into the agent's chat interface. That SSMS execution plan with the fat arrows? Paste it. The Activity Monitor showing a wall of blocked sessions? Paste it. The agent reads the image and incorporates what it sees into its analysis. It's not a substitute for actual plan XML or DMV output — text data is always more reliable — but during an incident when you need a quick read on what you're looking at, a screenshot gets you a faster first pass than exporting and formatting.

For follow-up evidence gathering, point the agent at Query Store, Extended Events, or the blocked process report — these give it the data it needs to move beyond initial triage into root cause.

For ongoing monitoring between incidents, [SqlServerAgMonitor](https://github.com/HannahVernon/SqlServerAgMonitor) tracks AG health continuously — the kind of baseline data that helps you spot drift before it becomes a 2 AM page.

---

Future satellite posts will dive deeper into specific troubleshooting scenarios — [CXPACKET/CXCONSUMER tuning](/ai-for-dbas/alter-dba-add-agent/), [deadlock root cause patterns](/ai-for-dbas/alter-dba-add-agent/), [memory grant analysis](/ai-for-dbas/alter-dba-add-agent/), and more.

**Next up:** [Incident Response: Root Cause Analysis with an AI Partner](/ai-for-dbas/incident-response-root-cause-analysis-with-an-ai-partner/) — using AI to build post-mortems and RCA documents from diagnostic data.


---

*Part of the [ALTER DBA ADD AGENT](/ai-for-dbas/alter-dba-add-agent/) series — [Previous: Legacy Code](/ai-for-dbas/understanding-unfamiliar-code-reverse-engineering-legacy-procedures/)*
