# What Can an AI Coding Agent Actually Do for a DBA?

In the [previous post](/ai-for-dbas/the-dbas-blind-spot-why-ai-coding-agents-are-coming-for-your-workflow/), I made the case that AI coding agents are fundamentally different from code completion — they don't just predict text, they execute work. But that's abstract until you see it mapped to the tasks you actually do every week.

This post is that map. Later posts go deep on each area. This is the overview.

## Writing and Reviewing T-SQL

You describe what you need — a query against `sys.dm_exec_query_stats` filtered by elapsed time, a CTE that ranks rows by partition — and the agent writes it. Not a generic template: T-SQL that follows [your coding standards](/tools/teaching-github-copilot-your-t-sql-coding-standards/), uses your actual column names, and handles the edge cases you'd forget until testing.

The review side is just as valuable. Paste a procedure you inherited and ask "what's wrong with this?" — the agent flags implicit conversions, deprecated syntax, and [business-logic bugs that SQL Server will happily compile and run](/t-sql/part-2-when-the-ai-code-reviewer-finds-bugs-the-optimizer-didnt/).

## Automating with PowerShell

Every DBA has a folder of PowerShell scripts in various states of completeness. The agent is useful here — and the iterative part is what matters. You start with "write a script that checks backup status across my CMS servers." The agent produces a first draft. "Also check for databases not in an AG" — it adds that. "Export to CSV with a timestamp" — done. What would take 45 minutes of `Get-Member` lookups becomes a five-minute conversation.

## Understanding Unfamiliar Code

This might be the single highest-value use case. You inherit a 2,000-line stored procedure with no documentation, dynamic SQL, and variable names like `@x1`.

Ask the agent to explain it in plain English. You get back a structured summary — what it does, what tables it reads and writes, what the parameters control, where the business logic hides. Ask "what would break if I removed this cursor?" and it traces the dependencies. The agent will read every line without skimming — though it may misinterpret complex business logic or miss runtime-dependent behavior that isn't visible in the static code.

## Troubleshooting and Incident Response

You're staring at [sp_WhoIsActive](https://github.com/amachanic/sp_whoisactive) output or a deadlock XML graph at 11pm. You know how to read these, but parsing them under pressure is slow and error-prone.

Feed the output to an agent. With enough context, it can often identify the blocking chain, trace a deadlock cycle, and suggest investigation paths. For wait stats, it separates signal from noise. When production goes sideways, it becomes a second pair of hands — you feed it error logs, ring buffer data, or `sys.dm_exec_requests` snapshots and get back a prioritized analysis instead of staring at raw output. It won't always get the diagnosis right — especially from partial data — but it accelerates the triage process.

Afterwards, it helps generate the post-mortem document from the diagnostic data you collected during the incident. In one case, I fed an agent two days of Spring Boot application logs from a REST API that had gone unresponsive. Within minutes, it identified HikariCP connection pool leak warnings from the day before the outage, traced them to a specific method that was holding a database connection open during an external HTTP call, and produced a root cause analysis with a recommended code fix — splitting the transaction boundary so the DB connection wasn't held during slow external I/O. That would have taken me hours of log parsing to piece together manually.

## Server Health Checks and Inventory

How current is your server inventory? If you're like most DBAs, there's a spreadsheet somewhere that was accurate six months ago. An agent can generate a comprehensive inventory script — versions, patch level, AG topology, memory configuration, backup status — and help you interpret the results across your entire fleet. It extends naturally to TLS audits, disk space monitoring, and configuration drift detection.

## Building Monitoring and Alerts

Long-running transactions, plan regressions, tempdb contention, log file growth — the agent generates monitoring queries with sensible thresholds, wires them up to Database Mail or operator alerts, and helps you tune the noise when they fire too often. For a more complete approach, [Erik Darling's PerformanceMonitor](https://github.com/erikdarlingdata/PerformanceMonitor) offers built-in AI integration that lets the agent query your monitoring data directly — we'll cover that in a [later post](/ai-for-dbas/ai-native-monitoring-performancemonitor-performancestudio-and-the-mcp-revolution/).

## Security Audits

"Find all orphaned users across every database on this instance." Most DBAs can write that script, but getting the edge cases right takes time. The agent handles the boilerplate — orphaned SQL logins, SID mismatches, empty principals with no permissions. It's equally useful for permission reviews, encryption audits, and compliance checklists. For more complex topics — contained database users, Entra ID principals, cross-database ownership chains — you'll still need to guide it with specific context about your environment.

## Migration Planning

Upgrading from SQL Server 2016 to 2025? The agent scans your codebase for deprecated features, identifies compatibility-level issues, and flags breaking changes in stored procedures. It won't make the architectural decisions for you — but it compresses the tedious code-scanning phase of migration planning. Use it to generate the findings list, then apply your own knowledge of what actually matters in your environment.

## Beyond Task Execution

Here's the part that doesn't get talked about enough: working with an agent teaches you things. When it writes a query using a DMV you haven't encountered, you learn that DMV. When it explains *why* it chose one approach over another, you absorb the reasoning. It's a useful reference — though you should verify its explanations against [Microsoft Learn](https://learn.microsoft.com/sql/), because it can be plausibly wrong about edge cases.

For senior DBAs, there's a team multiplier effect. Instead of being the sole source of tribal knowledge, you can point junior DBAs at the agent for routine questions while you focus on the judgment calls. The junior learns from the agent's explanations, and you stop answering the same question for the hundredth time. We'll dig into this in a [later post](/ai-for-dbas/the-ai-augmented-dba-team-mentoring-and-knowledge-transfer/).

---

That's the landscape. Every one of these areas gets its own deep dive later in the series, with real prompts, real output, and real tradeoffs.

**Next up:** [Getting Started: Your First Hour with GitHub Copilot CLI](/ai-for-dbas/getting-started-your-first-hour-with-github-copilot-cli/) — from zero to your first useful interaction.


---

*Part of the [ALTER DBA ADD AGENT](/ai-for-dbas/alter-dba-add-agent/) series — [Previous: The DBA's Blind Spot](/ai-for-dbas/the-dbas-blind-spot-why-ai-coding-agents-are-coming-for-your-workflow/)*
