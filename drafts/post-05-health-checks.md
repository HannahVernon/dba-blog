# Automating Server Health Checks and Inventory Scripts

Every DBA has a Monday morning ritual. Check backups. Check AG health. Scan for failed jobs. Glance at disk space. Maybe a quick look at error logs if something feels off.

Most of us have scripts for some of this. Few of us have scripts for all of it. And the scripts we do have tend to be cobbled together over years — some from blogs, some inherited from predecessors, some written at 2 AM during an incident and never cleaned up.

This is where an AI agent fits naturally: building, extending, and improving the health check scripts that keep your fleet healthy.

## The CMS Starting Point

If you're managing more than a handful of instances, you probably have a Central Management Server (CMS) or at least a server list somewhere. That's your starting point.

```
Write a PowerShell script that reads my registered servers from CMS
(using Get-DbaCmsRegServer from dbatools), connects to each one, and checks:
1. Any database with a full backup older than 24 hours (a starting point —
   production checks should also verify diff and log backup freshness
   relative to recovery model and RPO targets)
2. Any AG database not in a SYNCHRONIZED or SYNCHRONIZING state
   (suspended data movement, seeding failures, or disconnected replicas)
3. Any SQL Agent job that failed in the last 24 hours
Export the results to a CSV with columns: server_name, object_type, object_name,
issue, detected_at.
```

The agent gets you to a usable first draft faster than writing it from scratch. It handles the boilerplate — the `try/catch` around each server connection (because that one decommissioned instance in your CMS will break the whole loop), the CSV formatting, the date math.

A note on backup checks: "backup older than 24 hours" is a starting point, not production-grade monitoring. Real backup health checks need to evaluate expected backup types by database and recovery model, verify the backup was taken on the preferred AG replica, check for `CHECKSUM` usage, and — most importantly — include periodic actual restore tests. `RESTORE VERIFYONLY` is a quick sanity check, not a guarantee of recoverability.

Your first iteration won't be perfect.You'll say "also exclude databases named `tempdb` and `model` from the backup check" or "add a column for the AG role so I know if it's a primary or secondary." That iterative refinement is the natural workflow — describe, generate, refine.

**Try This Yourself:** Point the agent at your CMS or a text file with server names. Ask for a backup status check. See how close the first draft gets to what you'd write yourself.

## Building a Fleet Inventory

One of the most tedious DBA tasks is maintaining an accurate inventory. SQL Server version, edition, patch level, OS version, max server memory, compatibility levels — information that changes with every CU and every migration.

```
Write a PowerShell script using dbatools that connects to each server in
servers.txt and collects:
- SQL Server version, edition, and patch level
- OS version
- Total and available memory
- Max server memory setting
- Number of databases and total size
- Whether Query Store is enabled on each database
Export to an Excel file with one row per instance using ImportExcel module.
```

The agent generates the script, and you've got a living inventory document you can refresh whenever you need it. The real value is what happens next — you start asking follow-up questions:

- "Add a column showing how many CUs behind each instance is compared to my approved build list"
- "Flag any instance where max server memory equals the default (2147483647)"
- "Highlight instances still running SQL Server 2016 or earlier"

Each refinement takes seconds. Building this from scratch would take considerably longer.

A note on permissions: some inventory fields — OS version, total memory, available memory — require WMI/CIM remoting rights, not just SQL connectivity. Tell the agent what access you have so it generates the right approach.

## TLS Certificate Audits

I wrote [sql-cert-inspector](https://github.com/HannahVernon/sql-cert-inspector) because manually checking TLS certificates across a fleet is exactly the kind of tedious, error-prone work that should be automated. It connects at the raw TDS protocol level — no SQL authentication needed — and reports certificate details, health warnings, and Kerberos SPN diagnostics.

Where the agent shines is building the automation layer around it. Here's the kind of prompt that turns a single-server tool into a fleet-wide audit:

```
I have a tool called sql-cert-inspector.exe that accepts --server and --json flags
and outputs certificate details as JSON. Write a PowerShell script that:
1. Reads a list of SQL Server instances from servers.txt (one per line,
   supports server\instance format)
2. Runs sql-cert-inspector --server <instance> --json against each one
3. Parses the JSON output and collects: server name, certificate subject,
   expiry date, days until expiry, whether it's self-signed, and any warnings
4. Generates an HTML email body grouped by urgency:
   - RED: expired or expiring within 30 days
   - YELLOW: expiring within 90 days
   - GREEN: everything else
5. Sends the email via Microsoft Graph API using an app registration
Include error handling for unreachable instances and a summary count at the top.
```

The agent generates the complete script — the `foreach` loop, the JSON parsing with `ConvertFrom-Json`, the HTML table formatting, the Graph API call. You review it, test against a handful of instances, and you've got a weekly certificate renewal report that runs itself.

## AG Health at a Glance

If you're running Always On Availability Groups, you know the monitoring story is... incomplete. The built-in dashboard works for a single AG on a single cluster. For anything more complex, you need scripts.

```
Write a T-SQL script that queries sys.dm_hadr_availability_replica_states
and sys.dm_hadr_database_replica_states to show:
- Each AG and its replicas with synchronization health
- Any database not in a SYNCHRONIZED state
- Log send queue size and redo queue size
- Last hardened and last commit timestamps for lag estimation
Format the output as a clean result set I can run from SSMS.
Note: there is no single clean "commit latency" column in these DMVs —
derive lag estimates from timestamp differences and label them as estimates.
```

For a more comprehensive monitoring approach, check out [SqlServerAgMonitor](https://github.com/HannahVernon/SqlServerAgMonitor), which provides continuous AG health tracking. The agent can help you extend tools like this or build complementary scripts for your specific topology.

## Configuration Drift Detection

This is one of the most underrated health checks. You configure a new instance, set all the right `sp_configure` options, and six months later someone changes `cost threshold for parallelism` on one server and forgets to update the others.

```
Write a PowerShell script that captures sp_configure values from a baseline
instance (YOURBASELINE\SQL01) and compares them against every other instance
in my CMS. Report any setting where the value differs from baseline, showing
the baseline value and the actual value.
```

The agent builds the comparison logic, handles the multi-instance connection loop, and formats the drift report. You end up with a script that would have taken an hour to write and test — and you got it in a few minutes of prompting.

## Operationalizing: From Script to Scheduled Task

Getting a working script is half the battle. The other half is making it run reliably. Ask the agent to add:

- Error handling with notifications (Teams webhook, email via Graph API, or your preferred alerting channel)
- Logging with timestamps and instance counts
- Parameterized paths so nothing is hardcoded
- Timeout handling for unreachable servers

And a few operational hygiene items the agent won't think of unless you mention them:

- **Don't hardcode credentials.** Use Windows authentication, gMSA accounts, or the `SecretManagement` module. Specify the minimum SQL permissions needed — most health checks only require read access.
- **Be mindful of what goes into AI prompts.** Server names, topology details, certificate subjects, job names, and script contents are operational metadata. Use your organization's approved AI tooling and don't paste sensitive infrastructure details into consumer-grade AI services.
- **Define exclusions explicitly.** Dev instances, decommissioned servers, databases that legitimately don't need full backups — encode those decisions in a config file, not in your head.
- **Make backup checks AG-role aware.** A secondary replica in a readable AG has different backup expectations than a standalone instance.

The whole process — from "I need a backup audit" to "it runs every morning and notifies me if something's wrong" — can come together quickly. Getting to a reviewable draft is fast; production-hardening with proper error handling, edge cases, and security review still takes DBA judgment.

## When to Build vs. Buy

A quick word on judgment. The agent is great at generating scripts for *your specific environment*. But for well-understood, broadly applicable tasks, established community tools already exist — [dbatools](https://dbatools.io/), [Ola Hallengren's maintenance solution](https://ola.hallengren.com/), and [Erik Darling's PerformanceMonitor](https://github.com/erikdarlingdata/PerformanceMonitor) are battle-tested by thousands of DBAs.

Use the agent to *glue* these tools together, customize them for your environment, and fill the gaps they don't cover. Don't use it to rewrite what already works.

---

**Next up:** [PowerShell Automation: Backups, Maintenance, and AG Management](/ai-for-dbas/powershell-automation-backups-maintenance-and-ag-management/) — going deeper into the PowerShell scripts DBAs write every week.


---

*Part of the [ALTER DBA ADD AGENT](/ai-for-dbas/alter-dba-add-agent/) series — [Previous: Writing T-SQL](/ai-for-dbas/writing-t-sql-with-an-ai-partner/)*
