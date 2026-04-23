# AI-Assisted Pull Request Reviews for Database Code

If your team version-controls database code — stored procedures, migration scripts, deployment packages — you're probably doing pull request reviews. And if you're the senior DBA, you're probably the bottleneck.

PR review for database code is different from application code review. A missing semicolon in C# is a compile error. A missing semicolon in T-SQL might silently work today but already breaks specific constructs like CTEs and `MERGE` — and future enforcement may expand. The stakes are different, the expertise required is specialized, and the review queue is always too long.

This is where AI agents earn their keep.

## The First-Pass Review

The most immediate win is using the agent as a first-pass reviewer before the senior DBA looks at the PR. Not to replace the human review — to make it faster by catching the mechanical issues first.

```text
Review this deployment script for a production SQL Server 2025 environment.
Check for:
1. Missing rollback or roll-forward remediation plan
2. Implicit conversions in WHERE clauses
3. Inadequate error handling for the specific DDL being executed (note:
   TRY/CATCH alone isn't sufficient — check for XACT_ABORT, XACT_STATE,
   and whether the transaction pattern fits the operation)
4. Schema modification lock (Sch-M) risks on large or actively-written tables
5. Missing IF EXISTS/IF NOT EXISTS guards on CREATE/ALTER/DROP
6. Deprecated syntax for compatibility level 160
7. Schema-qualification on all object references
8. Semicolon termination
9. Drop/recreate side effects (lost permissions, extended properties, signatures)
Flag anything that would block deployment to production.
```

The agent returns a structured list of findings — the kind of checklist that takes a senior DBA fifteen minutes to work through manually. Most of the time, it catches the same things the human would catch: unguarded `DROP` statements, missing idempotency checks, hardcoded values that should be parameters.

The senior DBA still reviews. But instead of spending the first fifteen minutes on formatting and guard clauses, they start at the level where their experience matters: "Will this schema modification lock the Orders table during peak hours?" and "Does this index change invalidate the cached plans for the payment processor?" and "What does this do to AG redo lag?"

## Schema Change Reviews

Schema changes are where PR reviews matter most — and where they're hardest to get right. A column type change, a new index, a dropped constraint — these have blast-radius implications that aren't obvious from the diff alone.

```text
Review this ALTER TABLE migration script. The target table (dbo.Transactions)
has approximately 850 million rows and is actively written to by 12
application services. Evaluate:
1. Will this require a schema modification lock (Sch-M)? Is it a
   size-of-data operation or metadata-only?
2. Is there an online or resumable alternative?
3. What's the rollback path if this fails mid-execution? If a clean rollback
   isn't realistic, what's the roll-forward remediation?
4. Are there dependent views, computed columns, indexed views, or foreign keys
   that will break or need updating?
5. What's the impact on transaction log size, tempdb, and version store?
6. If this instance uses AGs, replication, CDC, or change tracking — what's
   the downstream impact?
7. Should this run during a maintenance window or can it run live?
```

The agent can identify questions to investigate and flag common risk patterns — especially if you've given it schema context through [custom instructions](/ai-for-dbas/custom-instructions-context/) or DDL files in the repository. It won't reliably assess actual production lock duration or log growth from a script alone — that depends on edition, indexes, compression, partitioning, LOB data, and active workload. But it surfaces the right questions to ask before you deploy.

**Try This Yourself:** Take a recent schema change PR and run it through the agent with the table's row count and write frequency. Compare the agent's risk assessment to what your team discussed in the actual review.

## Deployment Script Validation

Beyond the code itself, deployment scripts have structural requirements that are easy to miss in review:

```text
Check this deployment package (all .sql files in the deploy/ folder) for:
1. Execution order — are there dependency issues if scripts run in filename order?
2. Idempotency — can each script be safely re-run if a deployment fails partway?
3. Transaction boundaries — are related changes wrapped in a single transaction
   where appropriate?
4. Print/logging statements for deployment tracking
5. Pre-deployment validation checks (does the expected schema state exist?)
6. Post-deployment verification queries
```

The agent checks for the scaffolding around the actual changes — the parts that separate a script from a deployment. Missing idempotency guards are the most common catch: `CREATE PROCEDURE` without `IF NOT EXISTS` or `CREATE OR ALTER`, `INSERT` without a duplicate check, `ALTER TABLE ADD` without checking whether the column already exists.

One thing static review can't fully cover: **how the script runs matters.** `sqlcmd`, SSMS, DbUp, Flyway, SSDT/DacFx — they all handle `GO` batches, variable substitution, transaction boundaries, and stop-on-error behavior differently. If your deployment tooling matters (and it always does), tell the agent what runner you use.

## Data Migration and Backfill Scripts

This is where database PR reviews get real — and where AI review needs the most human oversight.

```text
Review this data migration script that backfills a new column on a table
with 200 million rows. Check for:
1. Batching strategy — is it processing in fixed-size chunks with commits?
2. Restartability — can it resume from where it stopped if interrupted?
3. Deterministic key ordering — is the batch window stable across retries?
4. Logging volume — will this blow out the transaction log?
5. Trigger side effects — will INSERT/UPDATE triggers fire on each batch?
6. Deadlock risk with concurrent application writes
```

The agent can evaluate the script's structure, but the operational questions — how long will this take, will it cause AG redo lag, can the log backups keep up — require environment context you'll need to provide or assess yourself.

## Permission and Security Changes

Permission changes in PRs deserve extra scrutiny. A `GRANT EXECUTE` seems harmless until you realize it gives a service account access to a procedure that can delete audit records.

```text
Review the permission changes in this deployment script. For each GRANT,
DENY, or REVOKE:
1. What principal is affected?
2. Does this follow least-privilege? Could the scope be narrower?
3. Are there any ownership-chaining implications?
4. Flag any GRANT WITH GRANT OPTION or CONTROL permissions
5. Flag any schema-level grants that deserve extra scrutiny
Note: effective permission analysis requires live environment metadata
(role memberships, server-level perms, module signing) — flag the intent
of each statement, not the cumulative result.
```

The agent can review the intent of GRANT/DENY/REVOKE statements and flag patterns worth questioning — broad schema-level grants, `WITH GRANT OPTION`, CONTROL permissions. It can't determine effective permissions from a script alone; that requires the actual database principals, role memberships, and server-level configuration.

## Building Review into Your Workflow

The highest-leverage approach is making AI review part of the PR process itself, not a manual step someone remembers to do:

1. **Pre-commit review:** Before pushing, run the deployment script through the agent locally. Fix mechanical issues before the PR is even created.

2. **Structured PR descriptions:** Use the agent to generate the PR description from the deployment scripts — it can summarize changed objects and risk categories from the code alone. For row counts, deployment windows, and environment-specific context, you'll need to add that yourself or feed it live metadata:

```text
Based on the SQL files in this changeset, generate a PR description that
includes:
- Summary of changes (what's being added/modified/removed)
- Tables affected (I'll add row counts manually)
- Risk level (schema change vs. data-only vs. permissions)
- Rollback or remediation steps
- Questions the reviewer should focus on
```

3. **Reviewer preparation:** When you're the reviewer, paste the PR diff and ask the agent to summarize it before you start reading. For large PRs with dozens of objects, this gives you a map before you dive into the details.

## What the Agent Misses

AI review catches syntax, structural, and pattern-based issues well. It's weak at:

- **Workload-specific performance impact** — it doesn't know your query patterns or peak hours unless you tell it
- **Cross-PR dependencies** — if another PR modifies the same table, the agent reviewing one PR won't know about the other
- **Organizational context** — "we never modify this table without notifying the billing team" isn't something an AI knows
- **Data-dependent behavior** — whether a migration will take 30 seconds or 30 minutes depends on data distribution the agent can't see
- **Existing plan cache implications** — dropping and recreating a procedure invalidates cached plans; the agent may not flag the performance cliff during recompilation storms
- **HA/DR downstream impact** — AG redo lag, replication latency, CDC capture lag, readable secondary query impact
- **Dynamic SQL and cross-database dependencies** — static analysis can't see into `EXEC(@sql)`, synonyms, or three-part names to linked servers
- **Runtime execution context** — `SET` options, `EXECUTE AS`, session settings, and deployment runner behavior all affect what the script actually does

The agent is a force multiplier for the reviewer, not a replacement. The senior DBA's experience — knowing which tables are hot, which service accounts matter, which deployment patterns have failed before — remains essential.

---

**Next up:** [The AI-Augmented DBA Team: Mentoring and Knowledge Transfer](/ai-for-dbas/ai-augmented-dba-team/) — how AI agents change mentoring, code review, and the role of the senior DBA.


---

*Part of the [ALTER DBA ADD AGENT](/ai-for-dbas/alter-dba-add-agent/) series — [Previous: Custom Instructions](/ai-for-dbas/custom-instructions-context/)*
