# Migration Planning: Compatibility Checks and Deprecated Features

SQL Server version upgrades are the kind of project that sounds straightforward and never is. You run your assessment tooling — Azure Arc migration assessment, the Database Experimentation Assistant, or even just your own diagnostic scripts — get a report with 200 findings, and then spend weeks figuring out which ones actually matter for your environment. The upgrade itself might take a weekend. The planning takes months.

An AI agent won't do the upgrade for you. But it can compress the planning phase by doing the tedious scanning, categorizing, and report generation that eats most of that time.

## Deprecated Feature Detection

Every SQL Server version deprecates features, and some removes them entirely. The challenge isn't finding the lists — Microsoft publishes those. The challenge is finding which ones *your* code uses.

For reference, here's the compatibility level matrix most DBAs keep in their heads:

| SQL Server Version | Compatibility Level |
|---|---|
| 2016 | 130 |
| 2017 | 140 |
| 2019 | 150 |
| 2022 | 160 |
| 2025 | 170 |

```text
Scan all stored procedures, functions, views, and triggers in this database
(from sys.sql_modules) and identify usage of:

DEPRECATED features (still functional but flagged for future removal):
1. Deprecated system tables (sysprocesses, sysobjects, syscolumns, etc.)
2. Deprecated syntax (non-ANSI outer joins *=, =*, GROUP BY ALL)
3. Deprecated data types (text, ntext, image)
4. String-based RAISERROR syntax (without parentheses)
5. Numbered stored procedures

DISCONTINUED/REMOVED features in SQL Server 2025:
1. Features actually removed from the engine in 2025
2. Functionality that will produce errors, not just deprecation warnings

Report the object name, object type, line number (approximate), the
specific feature found, and whether it's DEPRECATED or REMOVED.
```

The agent scans `sys.sql_modules` definitions and flags the matches, categorized by severity. It's not perfect — regex matching on procedure text has the same limitations we discussed in [the security post](/ai-for-dbas/security-audits-finding-what-you-missed/) with dynamic SQL scanning — but it catches the vast majority of deprecated usage in static code.

**What it can't catch:** deprecated features used in application-side SQL, SQL Agent job steps, SSIS packages, and dynamic SQL built at runtime. For runtime detection, use the **SQLServer:Deprecated Features** performance counters and the `deprecation_announcement` / `deprecation_final_support` Extended Events — these capture actual deprecated feature usage during normal workload execution, which is the only way to find what static code analysis misses.

Also check `sys.server_sql_modules` for server-level code: logon triggers, endpoints, and other objects that live outside user databases.

**Try This Yourself:** Pick your oldest database — the one everyone is afraid to touch — and run the deprecated feature scan. You'll almost certainly find `sysprocesses` references, old-style RAISERROR calls, and probably a few `text` columns that should have been converted to `varchar(max)` a decade ago.

## Compatibility Level Assessment

Changing the database compatibility level is where the real risk lives. It's not just about deprecated features — it affects query optimizer behavior, cardinality estimation, and execution plan selection.

```text
For a database currently at compatibility level 140 (SQL Server 2017) being
upgraded to compatibility level 160 (SQL Server 2022), analyze:
1. What query optimizer changes occur between these levels
2. Which cardinality estimator changes might cause plan regressions
3. What new features become available at compat level 160
4. Recommend a testing strategy for the compatibility level change,
   including how to use Query Store for A/B plan comparison
```

The agent generates a detailed breakdown of optimizer behavior changes by compatibility level. The real value is the **testing strategy** — it outlines a Query Store-based approach where you capture baseline plans at the old compat level, upgrade, compare, and force known-good plans for any regressions.

**Critical sequence:** enable Query Store and capture a meaningful baseline **before** changing the compatibility level. If you change compat level first, you have no before-picture to compare against. This is exactly the kind of step that's easy to skip under time pressure and painful to realize you missed.

## Breaking Change Analysis

Beyond deprecated features and compat level changes, there are outright breaking changes that will cause failures — not just regressions.

```text
I'm migrating from SQL Server 2016 to SQL Server 2025. Review the code
in this database for:
1. Features removed entirely (not just deprecated)
2. Syntax that will produce errors, not just warnings
3. System stored procedures that no longer exist
4. Changes to default behavior (e.g., string_split, UTF-8 collation handling)
5. Changes to DMV schemas that might break monitoring scripts
List each finding with severity: ERROR (will fail), WARNING (behavior change),
or INFO (deprecated but still functional).
```

The agent categorizes findings by severity, which is the part that takes the most time manually. "Will this actually break, or is it just a warning I can defer?" is the question you ask two hundred times during migration planning. Having that triage done for you — even approximately — saves days.

## Migration Runbook Generation

Once you know what needs fixing, you need a runbook. Every migration has the same structure — pre-checks, backup, execute, validate, rollback plan — but every environment has different specifics.

```text
Generate a SQL Server migration runbook for upgrading from SQL Server 2016
to SQL Server 2025 (in-place upgrade) for a 3-node Availability Group.
Include:
1. Pre-migration checklist (backups verified, compat issues resolved, etc.)
1. Step-by-step procedure with AG-specific rolling upgrade (secondaries first)
2. Validation queries to run after each node upgrade
3. Rollback procedure with clear distinction between pre-failover and post-failover scenarios
4. Post-migration steps (update compat level per Query Store plan, refresh statistics, validate)
Use conservative timeouts and include "stop and assess" checkpoints.
```

The agent generates a structured runbook you can customize for your environment. It won't know your specific server names, maintenance windows, or change management process — you fill those in. But the structure, the safety checks, and the order of operations follow well-documented best practices.

**AG rolling upgrade — the nuances that matter:**

The agent should recommend upgrading secondaries first, then failing over to an upgraded node. But the details are where mistakes happen:

- Upgrade **asynchronous DR replicas first**, then synchronous secondaries
- **Disable automatic failover** during the entire operation
- Only fail over to a secondary that is **fully synchronized** — check `synchronization_state` before pulling the trigger
- Readable secondary routing may need to be temporarily disabled during the upgrade
- Once the primary is on the newer version, it cannot ship logs to older-version secondaries — this is a one-way door
- **Rollback reality:** before you fail over to an upgraded replica, rollback is straightforward — just rebuild the upgraded secondary. After failover, the upgraded instance becomes the primary and databases start using the new version's internal format. At that point, "rolling back" means restore from backup, rebuild the AG, and reseed — not just failing back. Make sure your change management plan accounts for this.

If the agent doesn't include these caveats in a generated runbook, add them yourself. If it suggests upgrading the primary first, treat that as a red flag.

## Cloud Migration Considerations

If your migration target is Azure SQL Database or Managed Instance rather than a new on-premises SQL Server, the compatibility matrix gets more complex.

```text
Compare the feature support between our on-premises SQL Server 2016
database and Azure SQL Managed Instance. Flag:
1. Features we use that aren't supported in Managed Instance
2. Features that work differently (cross-database queries, linked servers, etc.)
3. SQL Agent capabilities that change
4. Authentication and networking differences
5. Features available in Managed Instance that we don't have today
```

The agent produces a gap analysis, but this is an area where you need to be precise — vague "it works differently" isn't helpful.

Some specifics for Azure SQL Managed Instance:
- **Cross-database queries** generally work on MI (unlike Azure SQL Database, where they don't)
- **Linked servers** are supported, but with restrictions on providers and authentication
- **SQL Agent** exists on MI, but some subsystems and job step types are limited
- **CLR assemblies** are supported with restrictions (SAFE only by default, EXTERNAL_ACCESS requires configuration)
- **Some `xp_` procedures** don't exist, and `xp_cmdshell` behavior differs
- **Filestream and FileTables** are not supported
- **Service Broker** works within a single MI, not across instances

**A strong caveat:** Azure feature support evolves constantly. The agent's training data has a cutoff date, and Azure SQL adds capabilities frequently. Always verify the agent's Azure compatibility claims against the [current Microsoft documentation](https://learn.microsoft.com/en-us/azure/azure-sql/managed-instance/transact-sql-tsql-differences-sql-server). Use the agent's output as a checklist of things to verify, not as a definitive compatibility matrix.

## Don't Forget the Adjacent Components

Database engine code is only part of the picture. Before any migration, inventory everything that touches the instance:

- **SSIS packages** — do they reference deprecated OLE DB providers or connection modes?
- **SSRS/SSAS** — do they need separate upgrade planning?
- **Replication and CDC** — version compatibility between publisher and subscriber
- **Client drivers** — ODBC, OLE DB, .NET SqlClient versions and TLS compatibility
- **Agent jobs** — CmdExec steps, PowerShell steps, SSIS package references
- **Linked servers and credentials** — provider compatibility on the new version
- **Vendor applications** — certified version support

Ask the agent to generate an inventory checklist for your specific environment. The more context you give it — server configuration, installed features, job definitions — the more thorough the checklist will be.

---

**Next up:** [AI-Native Monitoring: PerformanceMonitor, PerformanceStudio, and the MCP Revolution](/ai-for-dbas/ai-native-monitoring-performancemonitor-performancestudio-and-the-mcp-revolution/) — DBA tools built with AI integration from the ground up.


---

*Part of the [ALTER DBA ADD AGENT](/ai-for-dbas/alter-dba-add-agent/) series — [Previous: Security Audits](/ai-for-dbas/security-audits-finding-what-you-missed/)*
