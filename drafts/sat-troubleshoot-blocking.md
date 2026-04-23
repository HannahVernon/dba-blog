You get the alert: "Blocking detected, 47 sessions waiting." You open SSMS, run `sp_who2`, and get back a wall of rows that tells you almost nothing useful. Which session is the head blocker? What's it doing? How long has it been running? Is it safe to kill?

Blocking chain triage is one of those tasks that's conceptually simple — find the head, understand what it's doing, decide what to do — but operationally messy when you're staring at 50 rows of session data under pressure. An AI agent handles the parsing and pattern recognition; you handle the decision.

## Collecting the Right Data

The quality of the agent's analysis depends entirely on what you feed it. `sp_who2` is not enough. You need:

**sp_WhoIsActive.** Adam Mechanic's (now maintained by Erik Darling) [sp_WhoIsActive](https://github.com/ErikDarlingData/sp_WhoIsActive) is the gold standard. It shows blocking chains, wait types, running statements, and elapsed time in a single result set. If you don't have it installed, install it.

```sql
EXEC dbo.[sp_WhoIsActive]
    @get_outer_command = 1
  , @get_plans = 1
  , @find_block_leaders = 1
  , @sort_order = N'[blocked_session_count] DESC';
```

**sys.dm_os_waiting_tasks + sys.dm_exec_requests.** If you can't install sp_WhoIsActive (some shops have restrictions on third-party code), you can build a reasonable blocking chain view from DMVs:

```sql
SELECT
    wt.[session_id] AS [waiting_session_id]
  , wt.[blocking_session_id]
  , wt.[wait_type]
  , wt.[wait_duration_ms]
  , r.[command]
  , r.[status]
  , r.[total_elapsed_time] AS [elapsed_ms]
  , st.[text] AS [sql_text]
  , r.[database_id]
FROM sys.dm_os_waiting_tasks AS wt
INNER JOIN sys.dm_exec_requests AS r ON r.[session_id] = wt.[session_id]
CROSS APPLY sys.dm_exec_sql_text(r.[sql_handle]) AS st
WHERE wt.[blocking_session_id] IS NOT NULL
ORDER BY wt.[wait_duration_ms] DESC;
```

**sp_BlitzWho.** Erik Darling's [sp_BlitzWho](https://www.erikdarling.com/) provides similar data with additional context. Either tool works — the point is to capture the full blocking chain with query text, wait types, and elapsed times.

## Feeding a Blocking Chain to the Agent

Here's a realistic multi-level blocking chain. Session 87 is the head blocker, three sessions are directly blocked by it, and two more are blocked behind those.

```text
Here is sp_WhoIsActive output from a production SQL Server 2022 instance
experiencing a blocking chain. Analyze this and tell me:
1. Who is the head blocker and what is it doing?
2. How long has the head blocker been running?
3. What's the cascade — how many sessions are affected?
4. Is this a runaway query, an open transaction, or something else?
5. What should I do right now to resolve it?

session_id | status    | blocked_by | wait_type  | wait_ms  | sql_text (abbreviated)
-----------|-----------|------------|------------|----------|------------------------------------------
87         | sleeping  | NULL       | NULL       | NULL     | (last cmd: UPDATE dbo.Accounts SET ...)
142        | suspended | 87         | LCK_M_S    | 127433   | SELECT * FROM dbo.Accounts WHERE ...
198        | suspended | 87         | LCK_M_S    | 118221   | SELECT a.AccountID, ... FROM dbo.Accounts a
215        | suspended | 87         | LCK_M_IX   | 95102    | UPDATE dbo.Transactions SET ...
301        | suspended | 142        | LCK_M_S    | 89445    | SELECT ... FROM dbo.Accounts ...
318        | suspended | 198        | LCK_M_IS   | 72318    | SELECT TOP 100 ... FROM dbo.Accounts ...

Head blocker (87) outer command: BEGIN TRAN  UPDATE dbo.Accounts SET ...
Head blocker open_tran_count: 1
Head blocker login: CORP\jsmith
Head blocker host: DESKTOP-JSMITH7
Head blocker program: Microsoft SQL Server Management Studio - Query
Head blocker elapsed time: 00:14:22
```

## What the Agent Identifies

The agent processes this quickly and returns a structured analysis:

**Head blocker: Session 87.** Status is `sleeping` with `open_tran_count = 1` — this is the most dangerous pattern. The session ran an `UPDATE`, but it's sleeping, which means the query finished but the transaction was never committed. The `BEGIN TRAN` in the outer command confirms it: someone opened an explicit transaction in SSMS and walked away.

**Cascade impact.** Three sessions (142, 198, 215) are directly blocked by session 87. Two more (301, 318) are blocked transitively — they're waiting on sessions that are themselves waiting on 87. Total: 5 sessions blocked, the longest waiting over 2 minutes.

**Lock types tell the story.** Sessions 142 and 198 are waiting for shared (S) locks — they're just trying to read `dbo.Accounts`. Session 215 is waiting for an intent-exclusive (IX) lock — it's trying to write to `dbo.Transactions`, which likely shares a lock resource with the head blocker's uncommitted update. The head blocker is holding exclusive (X) locks from its uncommitted `UPDATE`.

**Resolution recommendation.** The agent suggests:

1. **Contact the user.** The login `CORP\jsmith` connected from `DESKTOP-JSMITH7` via SSMS. This is likely a developer or analyst who ran an ad-hoc update with an explicit transaction and forgot to commit. Call them first.
2. **If unreachable, kill session 87.** The transaction has been open for 14 minutes with a sleeping session — it's not going to commit on its own. Killing it will roll back the uncommitted update and release all blocked sessions.
3. **Prevent recurrence.** Suggest setting `SET LOCK_TIMEOUT` on the application connections, and consider implementing a monitoring job that alerts on sleeping sessions with open transactions older than a configurable threshold.

## What I Did

The agent's analysis was spot-on. I tried calling the developer first — no answer. After confirming with the application team that no critical batch process was running under that login, I killed the session:

```sql
KILL 87;
```

All five blocked sessions resumed immediately. The rollback was fast — the `UPDATE` had only modified a few hundred rows.

For the prevention side, I added an automated check using SQL Server Agent:

```sql
/* Alert on sleeping sessions with open transactions > 5 minutes */
IF EXISTS
(
    SELECT 1
    FROM sys.dm_exec_sessions AS s
    INNER JOIN sys.dm_tran_session_transactions AS t
        ON t.[session_id] = s.[session_id]
    WHERE s.[status] = N'sleeping'
      AND s.[open_transaction_count] > 0
      AND s.[last_request_end_time] < DATEADD(MINUTE, -5, SYSDATETIME())
      AND s.[is_user_process] = 1
)
BEGIN
    /* Send alert via Database Mail */
    EXEC msdb.dbo.[sp_send_dbmail]
        @profile_name = N'DBA Alerts'
      , @recipients = N'dba-team@example.com'
      , @subject = N'Sleeping session with open transaction detected'
      , @body = N'Check sys.dm_exec_sessions for sleeping sessions with open transactions.';
END;
```

## The Pattern That Matters

The real value here isn't that the agent told me to kill session 87 — I would have figured that out. The value is in the *speed* of the structured analysis. In 30 seconds, the agent:

- Identified the head blocker from a wall of session data
- Recognized the sleeping-with-open-transaction pattern
- Mapped the full cascade including transitive blocking
- Explained the lock types and what they meant
- Gave me a prioritized action list

During an active blocking event, those 30 seconds matter. Every minute session 87 stays open, more queries pile up behind it.

## Try This Yourself

1. Install sp_WhoIsActive if you haven't already — it's free and open source.
2. Next time you see blocking, capture the output with `@find_block_leaders = 1` and `@get_outer_command = 1`.
3. Paste the full output into your AI agent with the prompt structure above.
4. Compare the agent's analysis speed to your manual triage time — especially on complex chains with 10+ blocked sessions.
5. Build the sleeping-transaction monitoring job above and adjust the threshold for your environment.

The agent is fast at reading structured data. But the kill/don't-kill decision is yours — you need to understand what the transaction was doing and whether rolling it back will cause a bigger problem than the blocking itself.

For the full context on AI-assisted wait stats and blocking analysis, see [Post 8: Wait Stats, Deadlocks, and Blocking Chains](/ai-for-dbas/wait-stats-deadlocks-blocking/).

---

*Part of the [ALTER DBA ADD AGENT](/ai-for-dbas/alter-dba-add-agent/) series.*
