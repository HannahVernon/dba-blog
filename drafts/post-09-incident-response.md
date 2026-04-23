# Incident Response: Root Cause Analysis with an AI Partner

Last post we looked at using the agent to analyze wait stats, deadlocks, and blocking chains as individual diagnostic artifacts. This post is about putting those skills together during a real incident — the kind where you're juggling diagnostics, communication, and decision-making all at once.

## The Scenario

It's 9:47 AM. You get a Teams message: "Orders are timing out." Then another: "Reports aren't loading." Then your monitoring fires an alert for CPU above 90% on the primary production instance.

This is an idealized walkthrough — real incidents are messier, with multiple simultaneous symptoms and red herrings. But the workflow pattern holds even when the diagnosis isn't this clean.

## Step 1: Rapid Triage

You're already pulling up sp_WhoIsActive and checking wait stats. The difference is what you do with the output — instead of reading through it line by line while your phone buzzes, you paste it into the agent.

```text
Here's the current sp_WhoIsActive output and the top wait stats from the last
10 minutes. What's happening? Give me:
1. The most likely cause of the CPU spike
2. Whether this is a single bad query or a systemic issue
3. The top 3 things I should check next
```

The agent scans the output and highlights one session consuming significantly more CPU and logical reads than anything else — a request that's been running for six minutes against `dbo.OrderHistory` with a parallel plan. It flags this as worth investigating, but also notes there's blocking building behind it and a couple of memory grant waits that might be secondary effects.

That's thirty seconds of the agent's time versus five minutes of reading through the output yourself. And you still have attention left to scope the blast radius — is this one app, one stored procedure, or the whole instance? Check recent changes: deployments, statistics jobs, index rebuilds, failovers.

## Step 2: Digging Deeper

The agent identified a likely culprit. Now you need evidence.

```text
Write me a diagnostic query that shows:
1. The cached plan and request metadata for session 312 via
   sys.dm_exec_requests and sys.dm_exec_query_plan
2. The query hash and plan hash so I can correlate to Query Store
3. Recent Query Store runtime intervals for this query showing
   avg_duration, avg_cpu_time, and avg_logical_reads across plans
Don't include any personally identifying information in the output columns.
```

You run the query, paste the results back. The agent sees that average CPU time for this query's current plan is dramatically higher than the previous plan over recent intervals — a strong signal for plan regression. It suggests checking what changed: a statistics update, an index rebuild, parameter sensitivity, or a schema change that invalidated the prior plan.

This is the iterative diagnostic loop: the agent writes queries, you run them, the agent interprets the results. Each cycle takes a minute or two. The agent never touches production directly — you're the hands on the keyboard, it's the analyst reading over your shoulder.

## Step 3: The Fix

You've confirmed it's a plan regression. The agent suggests options:

```text
The query on dbo.OrderHistory regressed from a seek to a scan three days ago.
Options:
1. Force the last known good plan in Query Store
2. Add an OPTIMIZE FOR hint to the stored procedure
3. Update statistics on dbo.OrderHistory and hope the optimizer picks
   the better plan
4. Create a plan guide

Which would you recommend for an immediate fix during an active incident,
and which for a permanent fix afterward?
```

The agent recommends forcing the prior plan in Query Store as a controlled short-term mitigation — it's reversible and doesn't require a code change. But it's not magic: plan forcing affects future compilations, not the query already burning CPU right now. You may still need to kill the active session. And the forced plan needs to be valid for the current schema and index state — if something changed, the old plan might not be safe to force.

For the permanent fix, it suggests investigating the underlying cause: parameter sensitivity, stale statistics, data distribution drift, or a recent deployment that changed query behavior. `OPTIMIZE FOR` and plan guides are options, but they're band-aids — the real fix is understanding why the optimizer made a bad choice.

You force the plan, kill the active session, and watch CPU settle. Orders start flowing. In a well-instrumented environment with Query Store already enabled and healthy, this kind of triage can compress what would normally be a 20–30 minute diagnosis into something closer to 10–15 minutes. That's the best case — real incidents with multiple contributing factors take longer.

## Step 4: Building the Post-Mortem

This is where the agent really pays for itself. You have all the diagnostic data from the investigation — wait stats, sp_WhoIsActive output, query plans, Query Store history. Normally, writing the post-mortem takes longer than fixing the incident.

```text
Based on our investigation, write a post-mortem document with:
1. Timeline of events (first alert to resolution)
2. Root cause analysis (plan regression on OrderHistory query)
3. Immediate remediation (Query Store plan forcing)
4. Long-term fix recommendations
5. Prevention measures (monitoring for plan regressions)
6. Lessons learned
Use the diagnostic data I provided during our conversation.
```

The agent generates a structured post-mortem from the actual conversation history. The timeline is accurate because it was there for each step. The diagnostic evidence is included because you pasted it in. You review, adjust the tone for your audience, add the details the agent couldn't know (like "we checked the deployment log and found a stored procedure was redeployed Tuesday afternoon"), and submit it — in fifteen minutes instead of an hour.

One thing to add that the agent won't think of: **confirm recovery beyond a single metric.** CPU dropping isn't enough. Verify application latency, queue depth, error rates, and throughput. Watch for recurrence over the next hour. Don't declare victory because one chart improved.

I had a similar experience when the agent identified a [connection pool exhaustion issue](/ai-for-dbas/what-can-ai-agent-do-for-dba/) from application logs I pasted in. The post-mortem practically wrote itself because the entire diagnostic conversation was already captured.

## When to Stop Asking and Trust Your Gut

There's a moment in every incident where you know what's wrong but you can't fully articulate why. You've seen this pattern before. The metrics don't quite add up to a clean diagnosis, but your instinct says "it's the SAN" or "it's that deployment from last night."

Trust that instinct. The agent is good at systematic analysis, but it doesn't have your years of pattern recognition on *this specific environment*. If your gut says "check the deployment log," check the deployment log — even if the agent is still working through wait stats.

The agent is at its best when you use it as a fast research assistant during an incident, not as the decision-maker. You drive; it navigates.

---

**Next up:** [Security Audits: Finding What You Missed](/ai-for-dbas/security-audits-finding-missed/) — using AI to find the permission gaps, orphaned users, and compliance holes that manual reviews miss.


---

*Part of the [ALTER DBA ADD AGENT](/ai-for-dbas/alter-dba-add-agent/) series — [Previous: Wait Stats](/ai-for-dbas/wait-stats-deadlocks-blocking/)*
