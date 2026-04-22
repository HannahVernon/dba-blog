# Building Custom Monitoring Queries and Alerts

The [previous post](/ai-for-dbas/ai-native-monitoring-performancemonitor-performancestudio-and-the-mcp-revolution/) covered purpose-built monitoring tools with AI integration. This post is about the other side of monitoring — the custom queries and alerts that every DBA builds because no off-the-shelf tool covers everything your environment needs.

Every DBA has a collection of monitoring scripts. Some are polished and parameterized. Some are sticky notes on the monitor. Most sit in a folder called "scripts" or "monitoring" and haven't been touched since they were written. An AI agent can help you build better ones faster — and help you tune the ones that fire too often.

## Long-Running Transactions

This is one of the most common monitoring gaps. Your monitoring tool might catch long-running *queries*, but long-running *transactions* — where a session opened a transaction and then did something else (or nothing at all) — are harder to detect and often more damaging.

```
Write a monitoring query that detects open transactions older than
5 minutes. Include:
1. Session ID, login name, host name, program name
2. Transaction start time and duration
3. Whether the session is currently executing a request or idle
4. The last T-SQL command executed (from sys.dm_exec_connections)
5. Lock count held by the session
6. Log space used by the transaction (from sys.dm_tran_database_transactions)
Don't include any personally identifying information beyond login and host.
```

The agent generates a query joining `sys.dm_tran_active_transactions`, `sys.dm_tran_session_transactions`, `sys.dm_exec_sessions`, `sys.dm_exec_connections`, and a LEFT JOIN to `sys.dm_exec_requests` (to detect whether the session is actively executing or idle). In practice, you also need `sys.dm_tran_locks` aggregated by `request_session_id` for lock counts, and an aggregated join to `sys.dm_tran_database_transactions` by `transaction_id` to avoid duplicate rows across multiple databases.

The valuable detail most hand-written versions miss: log space consumption. A transaction holding 2 GB of log space is a different urgency level than one holding 2 KB — even if both have been open for the same duration.

One caveat: the "last T-SQL command" from `sys.dm_exec_connections` (`most_recent_sql_handle`) is the last batch seen on the connection, not necessarily the statement that opened the transaction. Use it as a clue, not proof.

## Alert Threshold Logic

The hardest part of monitoring isn't writing the query — it's setting the threshold. Too sensitive and you get alert fatigue. Too relaxed and you miss real problems.

```
I have a monitoring query that checks for blocking chains longer than
3 sessions. It fires about 20 times a day, mostly for blocking that
resolves within 30 seconds. Help me tune this:
1. Add a duration filter — only alert when the head blocker has been
   blocking for more than 60 seconds
2. Add a "consecutive check" pattern — only alert when the condition
   persists across two consecutive checks (30 seconds apart)
3. Add severity levels: WARNING at 60 seconds, CRITICAL at 5 minutes
4. Exclude known maintenance windows (Saturday 2-6 AM)
```

The agent transforms your noisy alert into a tiered monitoring system. The consecutive-check pattern is particularly useful — but it's not just query logic. It requires **persisted alert state**: a table keyed to the blocking condition, populated by each check, so the next check can compare. And be aware that if you're checking every 30 seconds and requiring two consecutive hits, your first alert won't fire until ~90 seconds — so your effective alerting latency is polling interval × required consecutive hits, not the threshold you set.

Also: if you need sub-minute polling, don't pretend SQL Agent makes this easy. A PowerShell service, a scheduled task, or a purpose-built monitoring platform is more appropriate than trying to run Agent jobs every 30 seconds.

**Try This Yourself:** Take your noisiest monitoring alert and paste it into the agent with "this fires too often — help me tune it." The suggestions usually involve a combination of duration thresholds, severity tiers, and time-based exclusions.

## Notification Integration

Many on-premises SQL Server shops still use Database Mail for alerts, and the agent can help you build properly formatted notification queries.

```
Write a stored procedure that sends a blocking alert via Database Mail.
Include:
1. An HTML-formatted email body with a table showing the blocking chain
2. The head blocker's current request text if active, or input buffer
   as fallback for sleeping sessions (plan handle may be NULL for idle blockers)
3. Color-coded severity (yellow for WARNING, red for CRITICAL)
4. A header showing the server name, alert time, and severity level
5. Parameter for the mail profile name and recipient list
6. Truncate query text to a safe length — don't ship full query text
   in alerts, as it may contain literals, customer data, or app secrets
```

The agent generates a complete stored procedure with HTML formatting. The HTML table output is significantly more readable than plain text — your on-call DBA gets a formatted blocking chain instead of a wall of text at 3 AM.

For environments that have moved beyond Database Mail — and modern teams increasingly route alerts through external systems — the agent can generate PowerShell wrappers that use `Invoke-RestMethod` to post to Teams, PagerDuty, or other webhook-based notification platforms. Keep the notification logic outside the SQL Server engine where possible; it's easier to maintain and doesn't expand the engine's attack surface.

## A Simple Monitoring Dashboard

You don't always need a commercial monitoring tool. For small environments or specific use cases, a PowerShell script that generates an HTML dashboard can be surprisingly effective.

```
Write a PowerShell script that:
1. Connects to a list of SQL Server instances (from a text file)
2. Checks each instance for: backup status, AG health, disk space,
   long-running queries, blocking, and job failures
3. Generates a single HTML file with a summary table
4. Color-codes each check: green (healthy), yellow (warning), red (critical)
5. Includes a timestamp and auto-refresh meta tag (5 minutes)
6. Saves the HTML to a network share where the team can access it
```

The agent generates a complete script — connection logic, diagnostic queries, HTML generation, and error handling. You customize the server list, thresholds, and network share path. The result is a lightweight status page for small environments or as a temporary stopgap while you justify proper monitoring infrastructure.

This is explicitly **not** a replacement for [PerformanceMonitor](/ai-for-dbas/ai-native-monitoring-performancemonitor-performancestudio-and-the-mcp-revolution/) or other purpose-built tools. It has no history, no real alerting pipeline, it goes stale if the job dies, and it doesn't scale to large estates. But it's better than nothing — and many DBAs live in the gap between "I have no monitoring" and "I've justified the time to set up proper monitoring" longer than they'd like to admit. For serious multi-server visibility, look at central collection into Grafana, Power BI, or a monitoring platform.

## Iterating on Alert Quality

One of the best uses of an AI agent in monitoring is the feedback loop. You write a monitoring query, deploy it, collect data about its behavior, and then ask the agent to help you improve it.

```
Here are the last 30 days of alert firings from my blocking monitor:
[paste CSV data]

Analyze these and tell me:
1. What percentage resolved within 60 seconds without intervention?
2. Are there recurring patterns — same time of day, same head blocker,
   same blocked query?
3. Based on this data, what threshold changes would reduce noise by 50%
   without missing any incident that required manual intervention?
```

The agent analyzes your alert history and recommends threshold adjustments backed by your actual data. This is the kind of analysis most DBAs know they should do but never have time for — and it's exactly the kind of tedious pattern recognition that AI agents excel at.

## When to Build Custom vs. Use a Purpose-Built Tool

Not every monitoring need requires custom code. Here's a rough decision framework:

- **Use a purpose-built tool** ([PerformanceMonitor](/ai-for-dbas/ai-native-monitoring-performancemonitor-performancestudio-and-the-mcp-revolution/), SentryOne, Redgate SQL Monitor) when you need continuous collection, historical trending, graphical analysis, or fleet-wide visibility
- **Build custom** when you need monitoring for a specific business process, a custom SLA metric, an unusual configuration, or a gap that no commercial tool covers
- **Start custom, graduate to tooling** — custom scripts are a great way to prove a monitoring need before investing in infrastructure

The AI agent helps with both approaches — it can help you evaluate commercial tools and it can help you build custom solutions. The goal is coverage, not tool loyalty.

## Production Hardening

AI is good at drafting monitoring scripts. The DBA is still responsible for join correctness, overhead, security, and false-positive control. Before deploying any AI-generated monitoring query to production, think about:

- **Alert suppression during maintenance windows** — don't wake people up during planned index rebuilds
- **State persistence** — consecutive-check patterns need history tables; design them to be small and self-cleaning
- **Alert cooldowns / dedup** — prevent the same condition from firing every polling cycle
- **Retention** — keep alert history for tuning analysis (the "iterate on alert quality" pattern above)
- **Least-privilege permissions** — monitoring queries should run under a dedicated login with only the read access they need
- **Polling overhead** — test your monitoring queries against busy production servers; some DMV queries are surprisingly expensive at scale
- **PII in alert payloads** — raw query text can contain customer data; truncate or hash by default
- **Runbook links** — every alert should link to remediation steps, not just describe the symptom

## Minimum Viable Monitoring Checklist

If you're building custom monitoring from scratch, here are the areas the agent can help you cover. You don't need all of these on day one, but this is the target state:

- Blocking chains with duration thresholds
- Long-running transactions with log space tracking
- Deadlock capture (Extended Events, not polling)
- Backup freshness and restore verification
- Agent job failures and long-running jobs
- AG health: redo queue, log send queue, synchronization state
- Transaction log growth, reuse wait type, VLF count
- Tempdb and version store pressure
- Storage latency from `sys.dm_io_virtual_file_stats`
- Severe or unusual wait type patterns
- Query Store plan regressions / plan instability

Each of these is a prompt away from a working draft. The agent drafts; you validate, harden, and deploy.

---

**Next up:** [Version Control and CI/CD: Unlocking What the Agent Can Actually Do](/ai-for-dbas/version-control-and-ci-cd-unlocking-what-the-agent-can-actually-do/) — why your database code in a repo is the key to unlocking the agent's full potential.


---

*Part of the [ALTER DBA ADD AGENT](/ai-for-dbas/alter-dba-add-agent/) series — [Previous: AI-Native Monitoring](/ai-for-dbas/ai-native-monitoring-performancemonitor-performancestudio-and-the-mcp-revolution/)*
