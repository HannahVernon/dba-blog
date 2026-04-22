# ALTER DBA ADD AGENT: Practical AI for Database Professionals

## Series Overview

A blog series on [SQLServerScience.com](https://sqlserverscience.com) helping traditional SQL Server DBAs understand how AI coding agents (primarily GitHub Copilot CLI, with Claude Code CLI referenced where relevant) can transform their day-to-day work. The series assumes readers who are experienced DBAs but new to AI coding agents — they've likely used ChatGPT and code-completion tools like IntelliSense or Redgate SQL Prompt, but haven't worked with agentic CLI tools that can read files, run commands, and iterate on solutions.

**Primary tool:** GitHub Copilot CLI (using Claude models like Opus 4.6)
**Secondary tool:** Claude Code CLI (mentioned where relevant — most concepts apply to both)

**Tone:** Practitioner-to-practitioner, first-person, problem-first framing with concrete examples. Real prompts and real (genericized) output. No AI marketing fluff.

## Editorial Guidelines

- **Keep posts concise and scannable.** DBAs are busy — no tomes. Each post should be a focused read, not War and Peace.
- **Practical and actionable first.** Social/professional impact is welcome but should serve the practical advice, not replace it.
- **"Try This Yourself" sections** woven naturally into posts where relevant — not a mandatory section, just a natural nudge.
- **Cross-references throughout.** Each post links to other posts in the series and to existing SQLServerScience.com posts where topics overlap.
- **Genericized examples.** Real prompts and real output, but scrubbed of anything exploitable (server names, credentials, internal topology).
- **Write in order.** Posts build on each other within themes, and earlier themes establish context for later ones.

---

## Theme 1: Foundations

### Post 1 — "The DBA's Blind Spot: Why AI Coding Agents Are Coming for Your Workflow"

**Goal:** The wake-up call post. Explain why AI coding agents represent a paradigm shift — not just fancier autocomplete — and why DBAs who ignore them risk falling behind.

**Key points:**
- The difference between code completion (SQL Prompt, IntelliSense, ChatGPT copy-paste) and agentic AI (reads your files, runs your commands, iterates on results)
- What "agentic" means in practice — the tool doesn't just suggest code, it understands your environment
- The disruption angle: junior DBAs who learn these tools will outpace seniors who don't; AI won't replace DBAs, but DBAs who use AI will replace those who don't
- The opportunity angle: mundane tasks that take hours become minutes — freeing you for the architecture and judgment work that actually matters
- Brief mention of what the series will cover

**Does NOT include:** Installation, setup, or hands-on examples. That's post 3.

---

### Post 2 — "What Can an AI Coding Agent Actually Do for a DBA?"

**Goal:** The "what" post — a top-down survey of every DBA task area where AI agents add value, with brief teasers of what's possible in each. Sets up the rest of the series.

**Key points:**
- T-SQL writing, review, and optimization (link to existing posts)
- PowerShell automation (backups, maintenance, AG failovers)
- Server health checks and inventory scripts
- Wait stats, deadlocks, and blocking chain analysis
- Security audits (orphaned users, permission sprawl)
- Documentation generation from schema
- Migration planning and deprecated feature detection
- Incident response and root cause analysis
- Monitoring query and alert creation
- Understanding unfamiliar stored procedures / reverse-engineering legacy code
- How AI helps the DBA *learn* — understanding the process the AI uses, not just the output
- How senior DBAs can use AI to mentor juniors more effectively (AI as a force multiplier for knowledge transfer)

**Format:** Survey-style with 1-2 paragraph descriptions per area, each ending with "we'll dig into this in [future post]."

---

### Post 3 — "Getting Started: Your First Hour with GitHub Copilot CLI"

**Goal:** Hands-on setup and first interaction. Get the reader from zero to productive in one post.

**Key points:**
- Prerequisites (GitHub account, Copilot subscription, terminal basics)
- Installation on Windows (the DBA's OS)
- First prompt: "Explain this stored procedure" — show a real interaction
- How the context window works — why the agent can "see" your files
- Custom instructions (link to the existing coding standards post)
- Common first-session mistakes (prompts too vague, not providing enough context)
- Quick win: ask it to document an undocumented stored procedure

---

## Theme 2: Daily Tasks

### Post 4 — "Writing T-SQL with an AI Partner"

**Goal:** Show how AI agents help write T-SQL faster and better — not by replacing the DBA's judgment, but by handling the boilerplate and catching mistakes.

**Key points:**
- Generating CREATE TABLE, ALTER, and migration scripts from natural language
- Writing complex JOINs and window functions — describe what you want, get working T-SQL
- The agent respects your coding standards (link to coding standards post)
- Code review: paste a proc, ask "what's wrong with this?"
- Refactoring legacy T-SQL (implicit joins → ANSI, missing semicolons, etc.)
- When NOT to trust the AI — always validate against actual schema (DDL reference)

### Satellite Posts — "T-SQL with AI" Mini-Series

**Candidate topics:**

- Implicit joins to ANSI — bulk refactoring with AI assistance
- Adding semicolons and fixing deprecated syntax across a codebase
- Writing window functions from natural language descriptions
- Generating CREATE/ALTER scripts from plain-English schema descriptions
- Dynamic SQL review — finding SQL injection risks and correctness issues
- Cursor-to-set-based rewrites — having the agent propose set-based alternatives
- Merge statement generation — describing upsert logic, getting a correct MERGE
- Temporal table queries — asking the agent to write FOR SYSTEM_TIME queries
- JSON/XML shredding — describing the document structure, getting working T-SQL
- Parameter sniffing diagnosis — feeding a plan to the agent and getting OPTIMIZE FOR recommendations

---

### Post 5 — "Automating Server Health Checks and Inventory Scripts"

**Goal:** Show how to use the agent to build PowerShell + T-SQL scripts for server inventory, health checks, and fleet-wide audits.

**Key points:**
- "Write a PowerShell script that connects to every server in my CMS and checks backup status"
- Iterating with the agent — first draft → "also check for databases not in an AG" → "export to CSV"
- Building a TLS certificate audit across a fleet (reference the sql-cert-inspector post)
- Generating SSMS-compatible reports
- Scheduling and operationalizing the scripts

### Satellite Posts — "Health Checks and Inventory" Mini-Series

**Candidate topics:**

- Backup status audit — comprehensive check across all instances with gap detection
- TLS certificate fleet audit — expiry, SANs, cipher suites, self-signed detection
- SQL Server version/patch inventory — build a living spreadsheet from your CMS
- AG topology mapping — replicas, sync state, listener configuration, preferred backup replicas
- Disk space monitoring — data/log/tempdb file growth trends with alerting
- Configuration drift detection — comparing sp_configure across instances to a baseline
- Database compatibility level audit — finding databases stuck on old compat levels
- Memory configuration review — max server memory, LPIM, lock pages in memory across a fleet
- SQL Agent job inventory — job status, schedules, failure history across instances
- Linked server audit — mapping all linked servers and their security configuration

---

### Post 6 — "PowerShell Automation: Backups, Maintenance, and AG Management"

**Goal:** Deep dive into PowerShell automation with AI assistance — the kind of scripting DBAs do weekly but rarely enjoy.

**Key points:**
- Backup scripts with proper error handling and logging
- Index maintenance scripts — show the agent generating Ola Hallengren-style logic
- AG failover runbooks — generating step-by-step scripts with safety checks
- The agent as a PowerShell tutor: "explain what this pipeline does"
- Building reusable modules from one-off scripts

### Satellite Posts — "PowerShell Automation" Mini-Series

**Candidate topics:**

- Backup automation — full/diff/log with error handling, logging, and notification
- Index maintenance — smart rebuild/reorganize with fragmentation thresholds
- AG failover runbook — pre-checks, failover, post-validation script
- Log shipping setup and monitoring via PowerShell
- Database refresh automation — restore prod to dev with data masking steps
- Automated DBCC CHECKDB scheduling with result capture and alerting
- SQL Agent job deployment — scripting and deploying jobs across instances
- Windows cluster health check — PowerShell for failover cluster validation
- Automated patching pre/post checks — capturing state before and verifying after
- PowerShell module packaging — turning one-off scripts into reusable, shareable modules

---

### Post 7 — "Understanding Unfamiliar Code: Reverse-Engineering Legacy Procedures"

**Goal:** One of the highest-value use cases — the DBA inherits a 2,000-line stored procedure and needs to understand it fast.

**Key points:**
- "Explain this stored procedure in plain English"
- "What tables does this proc read from and write to?"
- "Draw me the data flow" — getting a textual or mermaid diagram
- Finding hidden business logic in dynamic SQL
- Using the agent to generate test cases for legacy code you're afraid to touch

### Satellite Posts — "Legacy Code" Mini-Series

**Candidate topics:**

- Dynamic SQL dissection — having the agent trace what a proc actually executes at runtime
- Dependency mapping — "what breaks if I change this table?"
- Generating test data and test cases for undocumented procedures
- Cursor elimination — identifying cursors that can be replaced with set-based logic
- Trigger archaeology — understanding cascading trigger chains
- Linked server query analysis — tracing distributed queries through OPENQUERY/OPENROWSET
- ETL procedure documentation — mapping data flow through multi-step load procedures
- Job step analysis — understanding SQL Agent jobs with dozens of steps and conditional logic

---

## Theme 3: Troubleshooting

### Post 8 — "Wait Stats, Deadlocks, and Blocking Chains: AI-Assisted Diagnosis"

**Goal:** Show how AI agents can accelerate the troubleshooting workflow that every DBA does regularly.

**Key points:**
- Feeding wait stats output to the agent: "what's causing high CXPACKET waits on this server?"
- Deadlock XML analysis — paste a deadlock graph, get a plain-English explanation
- Blocking chain analysis — "here's sp_WhoIsActive output, what's happening?"
- Generating targeted fixes: index suggestions, query hints, configuration changes
- The agent as a second opinion during an incident

### Satellite Posts — "Troubleshooting with AI" Mini-Series

**Candidate topics:**

- CXPACKET/CXCONSUMER waits — diagnosis and MAXDOP tuning with AI
- Deadlock XML analysis — feeding graphs to the agent, getting root cause and fix
- Blocking chain triage — sp_WhoIsActive output to actionable steps
- PAGEIOLATCH waits — identifying I/O bottlenecks and missing indexes
- LCK_M_* waits — lock escalation analysis and resolution
- Memory grant analysis — identifying queries with excessive or insufficient grants
- THREADPOOL waits — worker thread exhaustion diagnosis
- RESOURCE_SEMAPHORE — query memory contention analysis
- Plan regression investigation — comparing before/after plans with AI assistance
- Spinlock contention — advanced diagnosis for high-concurrency workloads

---

### Post 9 — "Incident Response: Root Cause Analysis with an AI Partner"

**Goal:** Walk through a realistic incident scenario where the agent helps the DBA triage and resolve faster.

**Key points:**
- Scenario: production database suddenly slow
- Using the agent to analyze error logs, ring buffers, sys.dm_exec_query_stats
- Generating and interpreting diagnostic queries on the fly
- Building a post-mortem document from the investigation
- When to stop asking the AI and trust your gut

---

## Theme 4: Security and Compliance

### Post 10 — "Security Audits: Finding What You Missed"

**Goal:** AI-assisted security review — orphaned users, excessive permissions, missing encryption.

**Key points:**
- "Generate a script to find all orphaned users across my instances"
- Permission sprawl analysis — "who has sysadmin and why?"
- Checking for unencrypted connections, missing TDE, weak passwords
- Compliance report generation (SOX, PCI-DSS checklists)
- Using the agent to review your own security scripts for gaps

### Satellite Posts — "AI-Assisted Security" Mini-Series

Same pattern as the monitoring satellites — each topic becomes its own focused post linked back to Post 10.

**Candidate topics:**

- Orphaned users — finding and remediating across instances and versions (contained DBs, Azure AD, etc.)
- Permission sprawl — mapping who has sysadmin (and through what path: role membership, group nesting)
- Cross-database ownership chains — detecting and documenting
- TDE audit — which databases are encrypted, which aren't, certificate expiry tracking
- Connection encryption — identifying instances accepting unencrypted connections
- Service account hygiene — finding accounts with excessive permissions or stale passwords
- Row-level security review — auditing RLS policies for gaps
- SQL injection surface — using AI to scan stored procedures for dynamic SQL vulnerabilities
- Compliance checklists — generating SOX, PCI-DSS, HIPAA evidence from your actual configuration
- Azure AD / Entra ID integration audit — verifying external auth is configured correctly

---

## Theme 5: Advanced Topics

### Post 11 — "Migration Planning: Compatibility Checks and Deprecated Features"

**Goal:** Using AI to plan and de-risk SQL Server version upgrades and migrations.

**Key points:**
- "Scan this database for deprecated features in SQL Server 2025"
- Generating compatibility reports
- Identifying breaking changes in stored procedures, functions, and views
- Migration runbook generation
- Cloud migration considerations (Azure SQL DB, Managed Instance)

### Satellite Posts — "Migration with AI" Mini-Series

**Candidate topics:**

- Deprecated feature scan — comprehensive audit with AI-generated remediation plan
- Compatibility level impact analysis — what changes when you move from 130 to 160
- Data type migration issues — identifying implicit conversion risks in new versions
- Linked server and cross-database reference mapping for migration planning
- Azure SQL DB feasibility check — unsupported features, CLR, file tables, etc.
- Azure SQL Managed Instance migration — differences from on-prem that catch people
- Migration runbook generation — step-by-step with rollback procedures
- Post-migration validation — generating comparison queries to verify data integrity
- Performance baseline comparison — capturing before/after metrics across a migration

---

### Post 12 — "AI-Native Monitoring: PerformanceMonitor, PerformanceStudio, and the MCP Revolution"

**Goal:** Showcase Erik Darling's [PerformanceMonitor](https://github.com/erikdarlingdata/PerformanceMonitor) and [PerformanceStudio](https://github.com/erikdarlingdata/PerformanceStudio) as examples of DBA tools built with AI integration from the ground up — and what that means for the future of monitoring.

**Key points:**
- The problem with traditional monitoring: expensive per-server licensing, opaque analysis, data that leaves your network
- PerformanceMonitor: free, open-source, 32 collectors, real-time alerts — and a built-in MCP server with 51-63 read-only tools that let Claude or Copilot query your monitoring data directly
- PerformanceStudio: cross-platform execution plan analyzer with MCP integration — ask Claude "what's wrong with this plan?" and it reads the actual plan XML
- What MCP (Model Context Protocol) is and why it matters — it's how AI tools connect to your data sources without you copy-pasting output
- Show a real interaction: "What are the top wait types on my production server?" → AI queries PerformanceMonitor's MCP server → actionable answer
- How this changes the DBA's workflow: from "run a query, read output, interpret, decide" to "ask a question, get an analysis, validate and act"
- The broader trend: DBA tools that don't have AI integration will feel like tools without a GUI felt in 2005
- Link to the existing T-SQL optimization posts as examples of AI-assisted plan analysis

**Try This Yourself:** Install PerformanceMonitor Lite (nothing touches your server), connect it to a dev instance, then ask Claude Code about the collected data via MCP.

---

### Post 13 — "Building Custom Monitoring Queries and Alerts"

**Goal:** Using AI to create the custom monitoring that every DBA needs but never has time to build. Complements Post 12 — Post 12 covers purpose-built tools, this covers building your own.

**Key points:**
- "Build me a monitoring query for long-running transactions"
- Alert threshold logic — getting the agent to help set sensible defaults
- Integrating with Database Mail, operator notifications
- Building a simple monitoring dashboard with PowerShell + HTML
- Iterating on alert noise — "this fires too often, help me tune it"
- When to use a purpose-built tool (Post 12) vs. custom scripts (this post)

### Satellite Posts — "AI-Built Monitoring" Mini-Series

Each monitoring topic from Post 13 (and beyond) can become its own focused post: one problem, one AI-assisted walkthrough, one working solution. These are standalone posts that link back to Post 13 as the hub. Add them organically as the series progresses — no need to publish them all at once.

**Candidate topics** (each follows the pattern: problem → prompt → agent output → what I changed → final script):

- Long-running transactions — detecting and alerting
- Query plan regressions — identifying and notifying
- Tempdb contention — monitoring and diagnosing
- Log file growth — tracking and alerting before disk fills
- Blocking chain alerts — real-time notification with context
- High CPU queries — identifying top consumers with actionable detail
- AG health and latency — replica lag, synchronization state, failover readiness
- Backup failures and gaps — catching missed or failed backups across a fleet
- Index fragmentation — smart maintenance scheduling vs. blanket rebuilds
- Page life expectancy drops — detecting memory pressure before users notice
- Connection pool exhaustion — monitoring active connections vs. pool limits
- Deadlock frequency — trending and alerting on recurring deadlock patterns

**Editorial note:** These don't all need to exist before the series launches. They're the long tail — publish them as they're written, link them from Post 13 and the series index, and let the catalog grow over time. Each one is a self-contained "AI helped me build this" story that stands alone but adds depth to the series.

---

### Post 14 — "Version Control and CI/CD: Unlocking What the Agent Can Actually Do"

**Goal:** Show how AI agents become dramatically more useful when database code is version-controlled, and how the agent can help DBAs set up version control and CI/CD pipelines. Not a "you should use git" lecture — it's about what the agent can do with a repo versus without one.

**Key points:**
- The agent without a repo: working in isolation, you're the context bridge
- The agent with a repo: full schema context, cross-file dependencies, git history, PR review
- Getting your database code into a repo: agent-assisted schema export (dbatools, SSDT)
- What CI/CD looks like for database code: build validation, schema comparison, deployment preview
- How version control levels up earlier posts: code review, custom instructions, legacy analysis, migration
- Drift detection: comparing repo against live instances to catch undocumented changes
- Practical path: progressive adoption from git init to full CI/CD pipeline

---

### Post 15 — "Teaching AI Your Environment: Custom Instructions and Context"

**Goal:** Advanced usage — how to make the agent deeply effective by teaching it your specific environment.

**Key points:**
- Custom instructions files (copilot-instructions.md) — deep dive
- Providing schema context (DDL files, ERD descriptions)
- Environment-specific knowledge (server names, AG topology, backup policies)
- Building a "DBA knowledge base" that the agent can reference
- Team-wide standards — sharing custom instructions across a DBA team
- Link back to the coding standards post as a worked example

---

### Post 16 — "AI-Assisted Pull Request Reviews for Database Code"

**Goal:** Show how AI agents can accelerate and strengthen PR review workflows for database code — schema changes, deployment scripts, permission modifications, and migration packages.

**Key points:**
- AI as first-pass reviewer: catching mechanical issues (missing guards, deprecated syntax, standards violations) before the senior DBA reviews
- Schema change reviews: Sch-M lock implications, online/resumable alternatives, rollback/roll-forward paths, dependent object analysis
- Deployment script validation: execution order, idempotency, transaction boundaries, pre/post verification
- Data migration and backfill script review: batching, restartability, deadlock risk, log volume
- Permission and security change review: least-privilege checks, ownership chaining, WITH GRANT OPTION
- Building AI review into the workflow: pre-commit local review, structured PR descriptions, reviewer preparation
- What the agent misses: workload-specific impact, cross-PR dependencies, HA/DR downstream impact, dynamic SQL dependencies
- Referenced by Post 4 (code review section) and Post 17 (team code review workflows)

---

### Post 17 — "The AI-Augmented DBA Team: Mentoring and Knowledge Transfer"

**Goal:** How senior DBAs can use AI agents to level up their entire team.

**Key points:**
- Using AI to generate training materials from production scenarios
- Junior DBAs learning by watching the agent's reasoning process
- Code review workflows — AI as a first-pass reviewer before the senior DBA looks
- Building runbooks and documentation that stay current
- The senior DBA's new role: curator of AI context, not sole holder of tribal knowledge
- Career implications — where DBA skills are heading

---

### Post 18 — "How This Series Was Written: A DBA and an AI Walk Into a Terminal"

**Goal:** The meta post. Come clean about the process — this entire series was planned, outlined, drafted, and refined using the very AI coding agents it describes. Show the full behind-the-scenes process, including the prompts, the back-and-forth, the editorial decisions, and the mistakes.

**Key points:**
- Full transparency: "I used GitHub Copilot CLI to plan and write this series"
- The initial prompt and the questions the AI asked before starting
- How the series outline evolved through conversation — the decisions about tone, structure, naming
- The `process/` directory: raw conversation logs, editorial decisions, draft iterations
- What the AI got right on the first try, what needed heavy editing, and what I threw out entirely
- The irony of writing about AI disruption *with* AI — and why that's the point
- Lessons learned about working with AI on long-form content (not just code)
- The raw materials: link to or embed the full `dba-blog/` directory contents so readers can see everything

**Format:** This post includes or links to the full contents of the `dba-blog/` workspace directory, making the process fully transparent and reproducible.

**This directory (`C:\temp\dba-blog\`) is the artifact for this post.**

---

## Cross-Reference Map (Existing Posts)

| Existing Post | Series Posts That Should Link To It |
|---|---|
| "Can GitHub Copilot Optimize Your T-SQL?" | Posts 1, 2, 4, 12 |
| "Part 2: When the AI Code Reviewer Finds Bugs" | Posts 2, 4, 7, 12, 16 |
| "Teaching GitHub Copilot Your T-SQL Coding Standards" | Posts 2, 3, 4, 15 |
| "Inspecting SQL Server TLS Certificates" | Posts 2, 5 |
| "RegEx Replace in SSMS" | Post 4 |

## External References

| External Resource | Series Posts That Should Link To It |
|---|---|
| [PerformanceMonitor](https://github.com/erikdarlingdata/PerformanceMonitor) | Posts 2, 8, 12, 13 |
| [PerformanceStudio](https://github.com/erikdarlingdata/PerformanceStudio) | Posts 2, 8, 12 |
| [Erik Darling Data blog](https://erikdarling.com) | Post 12 |

## Author's Tools (Link Where They Fit Naturally)

These are the author's own open-source tools. Include only where genuinely relevant — this is not a product catalog.

| Tool | Repo | Natural Fit |
|---|---|---|
| sql-cert-inspector | [GitHub](https://github.com/HannahVernon/sql-cert-inspector) | Post 5 (fleet-wide TLS audit example) |
| SqlServerAgMonitor | [GitHub](https://github.com/HannahVernon/SqlServerAgMonitor) | Posts 5, 6, 8 (AG monitoring, health checks, troubleshooting) |
| GithubMarkdownViewer | [GitHub](https://github.com/HannahVernon/GithubMarkdownViewer) | Post 3 (viewing AI-generated documentation locally) |
| ai-security-audit | [GitHub](https://github.com/HannahVernon/ai-security-audit) | Post 10 (reusable AI prompts for security assessments) |
| pg-extract-schema | [GitHub](https://github.com/HannahVernon/pg-extract-schema) | Post 11 (schema extraction for migration planning) |
| pg-deploy | [GitHub](https://github.com/HannahVernon/pg-deploy) | Post 11 (incremental deployment scripts) |
| SQL-Server-Scripts | [GitHub](https://github.com/HannahVernon/SQL-Server-Scripts) | Posts 4, 7 (real scripts to analyze/refactor with an agent) |
| MVCTSQLJobScripter | [GitHub](https://github.com/HannahVernon/MVCTSQLJobScripter) | Post 6 (scripting SQL Agent jobs for automation) |

---

## Open Questions

*None remaining — ready to begin writing.*
