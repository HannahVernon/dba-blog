# AI-Native Monitoring: PerformanceMonitor, PerformanceStudio, and the MCP Revolution

Traditional SQL Server monitoring tools have a business model problem. They charge thousands of dollars per server per year, their analysis is opaque, and some of them phone your data home to their cloud. You pay for the privilege of looking at dashboards that tell you *something* is wrong without always helping you understand *why*.

Erik Darling's [PerformanceMonitor](https://github.com/erikdarlingdata/PerformanceMonitor) and [PerformanceStudio](https://github.com/erikdarlingdata/PerformanceStudio) take a different approach. They're free, open-source, and your monitoring data stays in your infrastructure. But the feature that makes them relevant to this series is their built-in MCP servers — they let AI agents query your monitoring data directly.

*Disclosure: Erik Darling is a friend and colleague. I'm not paid to recommend these tools. I recommend them because I use them and they're genuinely good.*

## What Is MCP and Why Should You Care?

MCP — Model Context Protocol — is a standard that lets AI tools connect to external data sources. Instead of you copy-pasting query output into the agent's chat window, MCP gives the agent direct read-only access to your data through a structured API.

Think of it this way: without MCP, working with an AI agent on a performance issue means you run a query, copy the output, paste it, wait for analysis, get a follow-up question, run another query, paste again... With MCP, the agent can query the data source itself. You go from being a copy-paste relay to having a conversation about the data.

This isn't a theoretical future — it's working today. Claude Code, Cursor, GitHub Copilot CLI, and other AI tools support MCP connections. The ecosystem is still maturing, but the core functionality works.

## PerformanceMonitor: 32 Collectors, 51+ MCP Tools

[PerformanceMonitor](https://github.com/erikdarlingdata/PerformanceMonitor) comes in two editions:

**Full Edition** installs a `PerformanceMonitor` database with 32 T-SQL collectors running via SQL Agent — wait stats, query performance, blocking chains, deadlock graphs, memory grants, file I/O, tempdb metrics, perfmon counters, and more. A separate dashboard app connects to view everything.

**Lite Edition** is a single desktop app that monitors remotely, storing data locally in DuckDB and Parquet files. It installs nothing on your server; most collection is read-only, though a few features may need additional server-side configuration depending on what you enable. This is the edition for quick triage, Azure SQL Database, locked-down servers, and consultants who need to assess a server without installing anything.

Both editions include a **built-in MCP server with 51-63 read-only tools** that AI agents can use to query your monitoring data. Real-time alerts (system tray, styled HTML email, webhooks), charts and graphs, CSV export, and a graphical plan viewer with a 30-rule PlanAnalyzer round out the package.

### MCP Security Boundaries

Before the "AI can query my monitoring data" alarm bells go off — the MCP server is **read-only**, binds to **localhost only**, and is **opt-in** (disabled by default). PerformanceMonitor's MCP exposes curated monitoring tools, not arbitrary SQL execution. PerformanceStudio's MCP exposes loaded plans and Query Store data, not raw database access.

One important distinction: the monitoring data itself stays local, but when you use an AI client (Claude Code, Cursor, etc.), the data the agent reads through MCP may be sent to the model provider as part of the conversation context. Review your AI client's data handling policies before using MCP with production telemetry.

### What This Looks Like in Practice

With PerformanceMonitor's MCP server connected to Claude Code, a conversation might go like this:

```
"What are the top wait types on my production server over the last 4 hours?
Which ones are actionable versus expected background waits?"
```

The agent queries PerformanceMonitor's MCP tools, retrieves the actual wait stats data, and analyzes it — distinguishing between waits that need investigation (high PAGEIOLATCH, unusual LCK waits) and waits that are normal for your workload (WAITFOR, LAZYWRITER_SLEEP, HADR_SYNC_COMMIT).

No copy-pasting. No screenshots. No running a separate query and reformatting the output. The agent reads the data and reasons about it in one step.

```
"Show me the blocking events from the last hour. Are any of these
recurring patterns or one-off incidents?"
```

The agent pulls blocking chain data from the monitoring database, correlates timestamps, identifies whether the same head blocker appears repeatedly, and flags the pattern. This is the kind of analysis that takes an experienced DBA fifteen minutes of scrolling through monitoring data — and the agent does it in seconds with access to the same underlying data.

## PerformanceStudio: Execution Plan Analysis with AI

[PerformanceStudio](https://github.com/erikdarlingdata/PerformanceStudio) is a cross-platform execution plan analyzer. It parses `.sqlplan` XML, identifies performance problems, suggests missing indexes, and provides actionable warnings — from the command line or a desktop GUI with a query editor, graphical plan viewer, and SSMS-style operator tooltips.

Its MCP server lets AI agents analyze execution plans directly:

```
"Load the plan I captured into PerformanceStudio and tell me why this
query is slow. What indexes would help?"
```

The agent reads the plan XML through MCP — once you've loaded it into the app or pulled it from Query Store — examining operator costs, row estimates vs actuals, memory grants, and parallelism. It provides specific recommendations backed by the actual operator data, not your description of it.

For DBAs who spend significant time in execution plan analysis — and that's most of us — this changes the workflow from "stare at the plan tree, click operators, mentally trace the data flow" to "ask a question about the plan and get an answer backed by the actual operator data."

## The Bigger Picture: MCP as a Standard

PerformanceMonitor and PerformanceStudio aren't the only tools adopting MCP. It's becoming a standard integration pattern — the way SQL Server tools connect to AI agents. As more DBA tools add MCP servers, the AI agent becomes a unified interface to your entire monitoring and diagnostic stack.

This is where the disruption angle from [Post 1](/ai-for-dbas/the-dbas-blind-spot-why-ai-coding-agents-are-coming-for-your-workflow/) starts to get concrete. As MCP adoption grows, DBA tools without AI integration will require more manual effort than tools that have it — the same way command-line-only tools lost ground to graphical interfaces. That's not a prediction about next month; MCP is still an emerging standard and the client ecosystem is still maturing. But the direction is clear.

## What This Doesn't Replace

To be direct about limitations:

- **MCP doesn't replace DBA judgment.** The agent analyzes data faster, but you still decide what to do about it. Wait stat analysis still needs workload context. Missing index suggestions are suggestions, not instructions.
- **The analysis is only as good as the collected data.** If collectors weren't running during the incident, if Query Store retention is too short, or if the monitoring database doesn't have enough history, the AI answer will be incomplete — and it may not tell you that.
- **MCP client support is still maturing.** Workflows aren't always polished. Expect some friction setting up connections and dealing with tool discovery.
- **PerformanceStudio analyzes plans well, but can't prove causality.** It can tell you a scan is expensive. It can't tell you whether fixing it will meaningfully improve the user experience.

That doesn't mean you should rip out your existing monitoring tomorrow. But when you evaluate new tools — or when existing vendors offer AI integration — MCP support should be on your checklist.

## Try This Yourself

1. Download [PerformanceMonitor Lite](https://github.com/erikdarlingdata/PerformanceMonitor/releases/latest) — the single-app edition that installs nothing on your server
2. Connect it to a **dev or test** instance (you can start with minimal read permissions; some features need additional rights depending on platform)
3. Let it collect data for an hour or two
4. Connect Claude Code to PerformanceMonitor's MCP server
5. Ask: "What are the top performance concerns on this server based on the collected data?"

Start on dev/test, not production. Get comfortable with the MCP workflow and understand what data flows to your AI client before pointing it at production telemetry.

---

For building your own custom monitoring queries and alerts (without a purpose-built tool), see [the next post](/ai-for-dbas/building-custom-monitoring-queries-and-alerts/). For the execution plan analysis techniques PerformanceStudio automates, see our earlier posts on [T-SQL optimization](/t-sql/can-github-copilot-optimize-your-t-sql-i-put-it-to-the-test/) and [AI code review](/ai-for-dbas/ai-assisted-pull-request-reviews-for-database-code/).

**Next up:** [Building Custom Monitoring Queries and Alerts](/ai-for-dbas/building-custom-monitoring-queries-and-alerts/) — for when you need monitoring that fits your specific environment.


---

*Part of the [ALTER DBA ADD AGENT](/ai-for-dbas/alter-dba-add-agent/) series — [Previous: Migration Planning](/ai-for-dbas/migration-planning-compatibility-checks-and-deprecated-features/)*
