# PowerShell Automation: Backups, Maintenance, and AG Management

Last post we used the agent to build health check scripts. Now let's go deeper into the PowerShell automation that DBAs do every week — backups, index maintenance, AG management, and the kind of scripting that always takes longer than you think it should.

## Backup Scripts That Actually Handle Errors

Every DBA has backup scripts. Most of them grew organically. Some of them silently fail and nobody notices until restore day.

```text
Write a PowerShell script using dbatools that backs up all user databases on
a given instance. Requirements:
- Full backups to \\backupshare\servername\databasename\ with timestamped filenames
- Use COPY_ONLY for AG secondary replicas
- Compress backups and verify with RESTORE VERIFYONLY (a useful quick check,
  though not a substitute for actual test restores)
- Wrap each database in try/catch — one failure shouldn't stop the rest
- Log results (success/failure, duration, backup size) to a CSV
- If any database fails, send a summary notification via Teams webhook
Use Windows authentication. No hardcoded credentials.
```

The agent generates a complete script with proper error isolation. The AG-role detection, the `COPY_ONLY` logic, the `RESTORE VERIFYONLY` call, the webhook formatting — all boilerplate the agent handles while you focus on whether the logic is right for your environment.

One thing to verify in the output: AG backup preference logic. The agent should use `sys.fn_hadr_backup_is_preferred_replica()` to determine whether the current replica should be taking backups — not just check whether a replica is primary or secondary. If it doesn't include this, ask for it explicitly. Getting this wrong means either duplicate backups or missing backups across your AG.

**Try This Yourself:** Take your current backup script and paste it into the agent. Ask it to add error isolation per database and a summary notification. Compare what it generates to what you have.

## Index Maintenance with Thresholds

For index maintenance, most shops use [Ola Hallengren's maintenance solution](https://ola.hallengren.com/) — and they should. But when you need custom logic, or you're building something to fill a gap Ola doesn't cover, the agent is useful for generating a first draft.

```text
Write a PowerShell script using dbatools that checks index fragmentation
on all user databases on a given instance. For each index:
- If fragmentation is between 10-30% and page count > 1000, reorganize
- If fragmentation > 30% and page count > 1000, rebuild online where the
  edition and index type support it — skip or defer indexes that can't
  rebuild online rather than defaulting to offline (which takes schema
  modification locks that block all queries on the table)
- Skip indexes under 1000 pages (fragmentation is meaningless at that size)
- Log every action to a table: dba.IndexMaintenanceLog with columns for
  database, schema, table, index, action, fragmentation_before, duration
```

The nuance the agent gets right: page count thresholds, edition-aware online/offline rebuilds, and the logging. The nuance it might get wrong: your specific maintenance windows, concurrency concerns, and whether `MAXDOP` should be constrained during rebuilds. Always review and test.

## AG Failover Runbooks

If you manage Availability Groups, you know that planned failovers aren't scary — but they're tedious if you do them carefully. The agent is good at generating runbook-style scripts with pre-checks and validation.

```text
Write a PowerShell failover runbook for an Always On AG. The script should:
1. Pre-checks: verify all replicas are healthy, all databases are SYNCHRONIZED,
   no active backup jobs running (check msdb job activity), and the target
   replica is in a SYNCHRONOUS_COMMIT state
2. If all pre-checks pass, prompt for confirmation before proceeding
3. Perform the failover using Switch-DbaAgReplica
4. Post-validation: verify the new primary is accepting connections, all databases
   are online, and the AG is healthy
5. Output a summary with before/after state
Implement as an advanced function with [CmdletBinding(SupportsShouldProcess)]
so -WhatIf runs pre-checks without actually failing over.
```

This is a script you'd normally build over several iterations across multiple maintenance windows, refining after each real failover. The agent gives you a solid first version that you then harden against your actual environment.

A word on safety: always test generated automation scripts against a single database or a single AG replica in non-production first. Adding `[CmdletBinding(SupportsShouldProcess)]` and using `-WhatIf` is a useful pattern for dry-run mode, but it only works if the code inside the function actually checks `$PSCmdlet.ShouldProcess()` before every destructive action. The agent sometimes generates the `CmdletBinding` attribute without wiring up the `ShouldProcess` checks inside the function body — verify that every `Switch-DbaAgReplica`, `KILL`, `DROP`, or state-changing call is gated behind `if ($PSCmdlet.ShouldProcess(...))`.

Also review the failover pre-checks: a production-grade runbook should verify AG queue sizes (log send queue and redo queue), cluster health, listener connectivity, and running backup/DBCC jobs — not just synchronization state.

One more: never let AI-generated scripts store credentials in plaintext, use `TrustServerCertificate` without understanding the implications, or skip certificate validation. These are easy shortcuts the agent will sometimes take if you don't tell it not to.

## The Agent as a PowerShell Tutor

This might be the most underrated use case. Not every DBA is a PowerShell expert — many of us learned just enough to get the job done and then stopped. The agent is helpful for explaining unfamiliar PowerShell.

```text
Explain what this pipeline does, step by step:

Get-DbaDatabase -SqlInstance $instance |
    Where-Object { $_.RecoveryModel -eq 'Full' -and $_.LastFullBackup -lt (Get-Date).AddDays(-1) } |
    Select-Object SqlInstance, Name, RecoveryModel, LastFullBackup |
    Sort-Object LastFullBackup |
    Export-Csv -Path ".\missing-backups.csv" -NoTypeInformation
```

The agent breaks it down: what each cmdlet does, what the `Where-Object` filter selects, why the sort order matters. More importantly, you can follow up with "how would I add error handling to this?" or "what if I want to run this against multiple instances?"

This is how you build PowerShell fluency — not by reading a book, but by getting line-by-line explanations of real scripts in your environment.

## From One-Off Scripts to Reusable Modules

Every DBA has a folder of scripts. Some are polished. Most are not. The agent can help you turn a collection of one-off `.ps1` files into a proper PowerShell module.

```text
I have these scripts in my DBA-Scripts folder:
- Backup-AllDatabases.ps1
- Check-AGHealth.ps1
- Get-IndexFragmentation.ps1
- Test-BackupRestore.ps1

Help me restructure these into a PowerShell module called DbaToolkit with:
- A proper module manifest (.psd1)
- Each script as an exported function
- Parameter validation on all functions
- Comment-based help with examples
```

The agent generates the module structure, converts scripts to functions with `[CmdletBinding()]`, adds parameter validation, and writes the manifest. You go from "a folder of scripts" to "a module I can `Install-Module` from a private feed" — and the cleanup work goes considerably faster than doing it by hand.

For SQL Agent job scripting specifically, [MVCTSQLJobScripter](https://github.com/HannahVernon/MVCTSQLJobScripter) handles the export side — scripting jobs across instances for version control and deployment. The agent can help you build complementary import/deployment scripts around it.

---

This post covered the automation patterns. Future satellite posts will go deeper into specific scenarios — [backup automation](/ai-for-dbas/alter-dba-add-agent/), [AG failover runbooks](/ai-for-dbas/alter-dba-add-agent/), [database refresh automation](/ai-for-dbas/alter-dba-add-agent/), and more.

**Next up:** [Understanding Unfamiliar Code: Reverse-Engineering Legacy Procedures](/ai-for-dbas/understanding-unfamiliar-code-reverse-engineering-legacy-procedures/) — using AI to make sense of the code you inherited.


---

*Part of the [ALTER DBA ADD AGENT](/ai-for-dbas/alter-dba-add-agent/) series — [Previous: Health Checks](/ai-for-dbas/automating-server-health-checks-and-inventory-scripts/)*
