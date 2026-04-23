# A Junior DBA's Field Guide to This Series: What Your Mentor Forgot to Explain

This series was written by a senior DBA, for senior DBAs. That's not a criticism — it's a design choice. The prompts assume you already know what wait stats are, that you've stared at a deadlock graph before, and that terms like "SARGability" don't send you to Google.

If you're 1-3 years into your DBA career, you can absolutely follow this series. But you're going to hit walls that more experienced readers won't even notice. This field guide maps those walls and gives you a path around them.

**Senior DBAs:** share this with your juniors *before* pointing them at the series. It'll save them from the two worst outcomes — quietly pretending they understand everything, or getting overwhelmed and bouncing off the material entirely.

## The Vocabulary Wall

The series assumes fluency in performance internals, Availability Group mechanics, and security architecture. Here's a quick orientation to the terms that come up repeatedly without much explanation.

**Performance internals:**
- **Wait stats** — SQL Server tracks *why* threads stop working. Every idle moment gets categorized. High `PAGEIOLATCH_SH` means threads are waiting for disk reads. High `LCK_M_X` means lock contention. The wait stats DMVs (`sys.dm_os_wait_stats`) are the starting point for most performance investigations.
- **CXPACKET / CXCONSUMER** — Parallelism waits. `CXCONSUMER` is usually fine (the coordinator thread waiting for workers). `CXPACKET` means one parallel worker finished and is waiting for the others — that's the one that signals skewed plans. Post 8 and the [CXPACKET satellite](/ai-for-dbas/ai-cxpacket-diagnosis/) go deep here.
- **Parameter sniffing** — SQL Server compiles a plan based on the first parameter values it sees. If those values are atypical, the plan is bad for everyone else. The [parameter sniffing satellite](/ai-for-dbas/ai-parameter-sniffing-diagnosis/) covers this, but you need to understand that plans are cached and reused.
- **Cardinality estimation** — The optimizer's guess at how many rows each operation will produce. Bad estimates cascade into bad join strategies, bad memory grants, and bad sort choices. When you see "estimated rows: 1, actual rows: 847,000" in an execution plan, that's a cardinality estimation failure.
- **Memory grants** — SQL Server pre-allocates memory for sorts and hashes before a query runs. If the grant is too small, the query spills to tempdb (slow). If it's too large, other queries can't get memory (also slow). The [memory grant satellite](/ai-for-dbas/ai-memory-grant-analysis/) covers diagnosis.
- **Plan cache** — Where SQL Server stores compiled execution plans for reuse. When the series talks about "plan cache pollution," it means too many single-use plans wasting memory.
- **SARGability** — "Search ARGument able." A predicate that can use an index seek. `WHERE LastName = 'Smith'` is SARGable. `WHERE LEFT(LastName, 3) = 'Smi'` is not — it forces a scan. When the agent suggests an index and the query wraps a column in a function, the index won't help.
- **Implicit conversions** — SQL Server silently converts data types when they don't match (e.g., comparing a `varchar` column to an `nvarchar` parameter). This often kills SARGability because the conversion wraps the column, preventing index seeks. The [implicit joins satellite](/ai-for-dbas/ai-convert-implicit-joins/) touches on this.
- **Execution plan analysis** — Reading the XML or graphical plan to understand what the optimizer chose. Fat arrows mean lots of rows. Warnings (yellow triangles) mean something went wrong. The most useful skill a junior DBA can develop is reading plans — not memorizing operators, but understanding data flow.

**Availability Group (AG) terminology:**
- **Synchronous commit** — The primary waits for the secondary to harden the log before acknowledging the commit. Slower, but zero data loss on failover.
- **Asynchronous commit** — The primary doesn't wait. Faster, but you can lose transactions on failover.
- **Redo queue** — Log records the secondary has received but hasn't yet replayed. A growing redo queue means the secondary is falling behind.
- **Send queue** — Log records the primary has hardened but hasn't yet sent to the secondary. A growing send queue means network or I/O issues.
- **Read-only routing** — Directing read-intent connections to secondary replicas. Requires configuration; it's not automatic.

**Security concepts:**
- **SID mapping** — A SQL login's Security Identifier. When you restore a database from one server to another, the database user's SID may not match any login on the new server — that's an orphaned user.
- **Orphaned users** — Database users with no matching login. The [orphaned users satellite](/ai-for-dbas/ai-orphaned-users/) covers detection and remediation.
- **CONTROL SERVER** — A server-level permission that's nearly as powerful as sysadmin but granted through the regular permission system. If the agent suggests granting this, be skeptical.
- **Cross-database ownership chaining** — When objects in different databases are owned by the same SID, SQL Server skips permission checks on the second object. It's a shortcut that creates security holes.

**Don't memorize all of this right now.** The point is to recognize these terms when they appear so you know what to look up — or what to ask the AI about.

### Try This: Use the Agent as a Glossary

When the series drops a term you don't know, use the AI agent itself to fill the gap. This is one of the safest things a junior DBA can do with an agent — there's no production risk in asking it to explain a concept.

```text
I'm reading about parameter sniffing in a blog series on AI for DBAs.
Explain parameter sniffing to me like I'm a DBA with one year of experience.
Include:
1. What it is in plain English
2. A concrete example with a stored procedure
3. Why it's hard to spot without knowing what to look for
4. What the symptoms look like in production (slow procedure
   that was fast yesterday)
```

The agent will give you a better explanation than most textbooks because you can keep asking follow-ups: "What does OPTIMIZE FOR UNKNOWN actually do?" or "Show me how to check if this procedure has multiple plans in the plan cache." This is the AI workflow that carries the least risk and the most learning value.

## The Production Safety Gap

The series says things like "verify the output" and "always review before running." That's correct advice, but it skips the *how* — and for a junior DBA, the how is everything.

**Rule zero:** Never run AI-generated code directly on production. Full stop. Not even SELECT queries you haven't reviewed — a `SELECT` with a bad join can consume enough CPU and memory to affect other workloads.

**What "testing" actually means:**

1. **Read the code line by line.** Not skimming — reading. If there's a clause you don't understand, ask the agent to explain it before you run anything.
2. **Run it on a non-production copy first.** If you don't have a dev/test environment, talk to your senior about setting one up. If you truly can't get one, at minimum run new queries inside an explicit transaction with `SET STATISTICS IO ON` and `SET STATISTICS TIME ON` so you can see the impact before committing.
3. **Check the execution plan before running.** In SSMS, hit Ctrl+L (estimated plan) or Ctrl+M (include actual plan). Look for table scans on large tables, warning icons, and estimated row counts that look absurd.
4. **Know what rollback means.** When Post 9 talks about killing a session, understand that `KILL <spid>` doesn't just stop the query — it rolls back any open transaction, and that rollback can take longer than the original operation. Killing a 30-minute `DELETE` can trigger a 45-minute rollback.

**Specific areas that should make your palms sweat:**

| Action | Why It's Dangerous | What to Do Instead |
|--------|-------------------|-------------------|
| `KILL <spid>` | Triggers rollback; can make things worse | Ask your senior. Check the transaction log usage first. |
| `sp_configure` changes | Server-wide settings; some require restart | Document the current value *before* changing. Know which settings are `advanced`, which need `RECONFIGURE WITH OVERRIDE`, and which need a service restart. |
| AG failover runbooks | Wrong sequence = data loss or split-brain | Never execute without a senior DBA present. Read the [failover satellite](/ai-for-dbas/ai-ag-failover-runbook/) for understanding, not for solo execution. |
| Changing compatibility level | Enables new cardinality estimator; plans change | Can cause widespread performance regression. Test with Query Store's A/B comparison mode. |
| `DROP USER` / `ALTER USER` | Can break application access instantly | Verify who uses that login. Check `sys.dm_exec_sessions` for active connections. Have a rollback script ready. |
| Index rebuilds during business hours | Locks tables (unless ONLINE), fills the log | Use ONLINE rebuilds if your edition supports them. Check table size and log space first. Schedule during maintenance windows. |

**The meta-skill here is learning to ask:** "What's the blast radius if this goes wrong?" If the answer is "it affects one query," the risk is low. If the answer is "it affects every connection to this server," that's a senior-DBA conversation.

## The Confidence Calibration Problem

This is the hardest section to write, because the honest answer isn't comfortable.

The AI agent produces output that *reads* like expert analysis. Clear explanations, specific recommendations, formatted results. And a lot of it is correct. But the hardest part for a junior DBA isn't running the code — it's knowing whether the analysis is *right*.

You can't verify what you don't understand yet. That's not a character flaw; it's where you are in your career.

**Areas where you're most likely to be fooled:**

- **Execution plan changes.** The agent says "this plan is better because it uses a seek instead of a scan." That's often true. But sometimes a scan on a small table is faster than a seek plus key lookup. Without experience reading hundreds of plans, you can't always tell "better" from "just different."
- **Security audit completeness.** The agent lists permission issues it found. But can you tell if it *missed* something? Did it check server-level permissions, database-level permissions, *and* object-level permissions? Did it look at role membership chains? A clean-looking report isn't the same as a thorough audit.
- **Deadlock root cause.** The agent reads the deadlock XML and says "Process A holds a lock on Table X and needs Table Y, while Process B has the opposite." That's the deadlock *description*, not necessarily the root cause. The root cause might be a missing index, a transaction held open too long, or a bad application retry pattern. Without experience, you'll accept the description as the diagnosis.
- **Migration feasibility.** The agent says "this database is ready to migrate to Azure SQL Database." But did it account for cross-database queries, linked servers, CLR assemblies, Service Broker, or the fourteen other features that Azure SQL Database doesn't support? The [deprecated features satellite](/ai-for-dbas/ai-deprecated-feature-scan/) and [Azure feasibility satellite](/ai-for-dbas/ai-azure-sql-feasibility/) cover the detection — but you still need to judge the severity of each finding.

**What to do about this:**

The answer isn't "wait until you're senior." It's "be transparent about what you don't know."

When the agent gives you an analysis and you think "that looks right," apply this test: *Could I explain to my senior DBA why this is the correct answer?* Not just repeat what the agent said — actually explain the reasoning. If you can't, you have two good options:

1. Ask the agent to explain its reasoning step by step, then check that reasoning against documentation (Microsoft Docs, not blog posts, for official behavior).
2. Bring it to your senior: "The agent suggested X because of Y. Does that reasoning hold?"

Option 2 is the faster way to build the judgment you need. And most senior DBAs will be impressed that you're asking, not annoyed.

## The Community Knowledge Assumption

The series references tools and people by name, assuming you know the context. If you're newer to the SQL Server community, here's a quick orientation.

**People:**

- **Erik Darling** — Performance tuning specialist. His [sp_PressureDetector](https://github.com/erikdarlingdata/DarlingData) is one of the first tools you should run when a server is under pressure. His [PerformanceMonitor](https://github.com/erikdarlingdata/PerformanceMonitor) is referenced in the monitoring posts. His blog ([erikdarling.com](https://erikdarling.com)) is one of the best resources for understanding how SQL Server *actually* behaves vs. how the documentation says it should.
- **Adam Machanic** — Created [sp_WhoIsActive](http://whoisactive.com), which is the de facto standard for "what's running on my server right now." If your servers don't have it installed, fix that. It shows running queries, wait types, blocking chains, and tempdb usage in one result set. You'll see it referenced throughout the troubleshooting posts.

**Tools and scripts:**

- **Ola Hallengren's maintenance scripts** ([ola.hallengren.com](https://ola.hallengren.com)) — The gold standard for backup, index maintenance, and integrity checking in SQL Server. Open source, battle-tested, runs on thousands of production servers. If your team doesn't use these or something equivalent, ask why.
- **dbatools** ([dbatools.io](https://dbatools.io)) — A PowerShell module with 500+ commands for SQL Server administration. Migration, backup/restore, AG management, security audits — it covers almost everything. The PowerShell posts in this series often reference dbatools commands. Install it: `Install-Module dbatools`.
- **sp_Blitz suite** — A set of scripts originally created by Brent Ozar and now actively maintained by Erik Darling. Includes tools for configuration checks (`sp_Blitz`), index problems (`sp_BlitzIndex`), plan cache analysis (`sp_BlitzCache`), deadlock analysis (`sp_BlitzLock`), and first-responder troubleshooting (`sp_BlitzFirst`). Free, open source, and widely used for health checks.
- **SSDT / DacFx** — SQL Server Data Tools provides a project-based way to manage database schemas in source control. DacFx is the underlying framework. Post 14 on version control assumes familiarity with this approach. If you've only ever made changes directly in SSMS, the concept of a "database project" will feel unfamiliar — but it's how teams manage database schema at scale.

You don't need to master all of these before reading the series. But when a post says "run `sp_WhoIsActive` and paste the output," you should know what that tool is and where to get it.

## A Junior's Safe Starting Path

Not all posts are the same difficulty or carry the same risk. Here's a reading order designed for someone who's still building their SQL Server foundation.

### Phase 1: Context and Setup (Low Risk)

Start here. These posts explain what AI agents are, what they can do, and how to set one up. No production changes involved.

- **Post 1** — [The DBA's Blind Spot](/ai-for-dbas/the-dbas-blind-spot/) — What coding agents are and why they matter
- **Post 2** — [What Can an AI Agent Do for a DBA?](/ai-for-dbas/what-can-ai-agent-do-for-dba/) — A survey of use cases
- **Post 3** — [Getting Started with GitHub Copilot CLI](/ai-for-dbas/getting-started-copilot-cli/) — Installation and first prompts
- **Post 15** — [Custom Instructions](/ai-for-dbas/custom-instructions-context/) — Setting up persistent context (skip here early — it makes every other post work better)

### Phase 2: Training Wheels (Low-Medium Risk)

These posts involve generating and reviewing code, but the focus is observation and learning — not deploying changes.

- **Post 4** — [Writing T-SQL with an AI Partner](/ai-for-dbas/writing-tsql-with-ai-partner/) — Start with the code generation and review sections. Don't deploy anything to production; practice on a test database.
- **Post 5** — [Health Checks and Inventory](/ai-for-dbas/health-checks-inventory/) — Generating diagnostic queries. These are read-only by design — `SELECT` from system views, no modifications. Safe to run, valuable for learning what's on your servers.

### Phase 3: Intermediate (Medium Risk)

Now you're using the agent for real work, but in areas where the consequences of mistakes are limited.

- **Post 7** — [Reverse-Engineering Legacy Procedures](/ai-for-dbas/reverse-engineering-legacy-procedures/) — Use the agent to *understand* code, not rewrite it. This is pure analysis — paste in a stored procedure and ask for an explanation. Zero production risk, high learning value.
- **Post 14** — [Version Control for DBA Work](/ai-for-dbas/version-control-cicd/) — Source control fundamentals. If you're not already using Git for your scripts, this post will change your workflow.

### Phase 4: Advanced (Higher Risk — Read and Learn)

Read these to understand the concepts. Practice on test environments. Don't act on the output in production without supervision.

- **Posts 8-9** — [Wait Stats and Deadlocks](/ai-for-dbas/wait-stats-deadlocks-blocking/) and [Incident Response](/ai-for-dbas/incident-response-root-cause/) — You need these skills, but applying them in production requires judgment you're still developing.
- **Post 6** — [PowerShell Automation](/ai-for-dbas/powershell-automation/) — Automating server tasks. Powerful but dangerous if the automation has a bug.
- **Posts 10-11** — [Security Audits](/ai-for-dbas/security-audits-finding-missed/) and [Migration Readiness](/ai-for-dbas/migration-planning-compatibility/) — Sensitive areas where incomplete analysis has real consequences.

### Phase 5: Senior-Supervised Only

These tasks have production impact and require experienced judgment. Read the posts for understanding. Execute only with a senior DBA present.

- AG failover execution (satellite: [AG Failover Runbook](/ai-for-dbas/ai-ag-failover-runbook/))
- Production incident response during active outages
- Compliance evidence generation (satellite: [Security Compliance](/ai-for-dbas/ai-compliance-checklists/))
- Any `sp_configure` changes on production instances
- Index strategy changes on high-transaction systems

## What Your Senior DBA Wants You to Know

This section is for the seniors. If you're handing this series to a junior team member, here's how to make it productive instead of dangerous.

**Pair on the first few AI sessions.** Watch what they accept versus what they question. The patterns you see in those first sessions will tell you where the gaps are. If they accept everything without pushback, they need more foundation before they're ready to use the agent independently.

**Set explicit guardrails.** Don't leave it vague. Say exactly what they can and can't do:

```text
✅ "You can use the AI to generate diagnostic queries and run them on DEV."
✅ "You can use it to explain stored procedures and review code."
✅ "You can use it to draft scripts, but I review before anything touches TEST or above."
❌ "Don't run AI-generated DDL on any environment without a review."
❌ "Don't use it for security changes — bring those to me."
❌ "Don't kill sessions on production based on AI advice — come find me first."
```

**The AI makes juniors sound more confident than they are.** This is both a feature and a risk. The feature: they can draft scripts, write documentation, and prepare analyses that would have taken years of experience to produce manually. The risk: the output looks polished whether the person behind it understands the material or not. Ask questions that test understanding, not just output quality.

**Use the satellite posts as supervised exercises.** The satellite posts are designed as focused, single-topic walkthroughs. They make excellent homework: assign a satellite, have the junior follow the prompts on a test environment, and review the results together. The [backup audit](/ai-for-dbas/ai-backup-status-audit/), [version inventory](/ai-for-dbas/ai-version-patch-inventory/), and [implicit join conversion](/ai-for-dbas/ai-convert-implicit-joins/) satellites are good starting assignments — low risk, high educational value.

**The goal is building judgment, not just productivity.** An AI agent can make a junior DBA 3x faster at generating scripts. But if they can't evaluate those scripts — if they're just a conduit between the AI and your production environment — that's not productivity. It's risk with extra steps. The series helps with the "how to use the tool" part. Your job is the "when to trust the output" part.

## The One Rule

If you take nothing else from this field guide, take this:

When the AI gives you an answer and you think "that looks right," ask yourself one question: **Could I explain to my senior DBA *why* this is right?**

Not repeat the answer. Not show them the output. *Explain the reasoning.*

If you can't, you're not ready to run it in production. That's not a judgment on you — it's a signal that you need to learn one more thing before proceeding. And the fastest way to learn that thing is to ask the agent to explain its reasoning, then verify that reasoning with documentation or a colleague.

```text
You just suggested adding a nonclustered index on OrderDate with
CustomerID as an included column. Walk me through why:
1. Why OrderDate as the key column and not CustomerID?
2. What queries would this index help vs. not help?
3. What's the write overhead of this index on a table that gets
   5,000 INSERTs per hour?
4. How would I monitor whether this index is actually being used
   after I create it?
```

The agent will answer all of these. Then you can take that reasoning to your senior and have a real conversation about whether the index makes sense — not "the AI said so" but "here's the tradeoff and here's why I think it's worth it."

That's the difference between using AI as a crutch and using AI as a learning accelerator. You belong in this series. You belong in this career. Just bring your curiosity and your healthy skepticism, and ask for help when the material exceeds your experience. That's not weakness — it's how every senior DBA you know got to where they are.

---

*Part of the [ALTER DBA ADD AGENT](/ai-for-dbas/alter-dba-add-agent/) series.*
