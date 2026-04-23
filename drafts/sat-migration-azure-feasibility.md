"Can we move this to Azure?" is the question that launches a thousand spreadsheets. The honest answer is almost always "it depends" — and the gap between "yes, this is a straightforward lift-and-shift" and "this will take six months of refactoring" often comes down to a handful of features that don't exist in Azure SQL Database or behave differently in Managed Instance.

An AI agent can't make the business case for you, but it can do the tedious feature-by-feature compatibility assessment that usually takes days of cross-referencing documentation. Give it a clear picture of what your database uses, and it'll tell you where the blockers are.

## The Problem

You have an on-premises SQL Server 2019 database supporting a line-of-business application. Management wants to move to Azure. You need to answer three questions quickly: Can it go to Azure SQL Database? Can it go to Managed Instance? What breaks either way?

The challenge is that Microsoft's compatibility documentation is spread across dozens of pages, and the answer changes depending on *which* Azure SQL platform you're targeting. Azure SQL Database and Azure SQL Managed Instance have very different feature support matrices.

## The Prompt

```text
I need to assess whether an on-premises SQL Server 2019 database can
migrate to Azure. Here's what the database uses:

- 340 GB data, 45 GB log, FULL recovery model
- 12 cross-database queries joining to a shared reference database
- 3 CLR assemblies (SAFE permission set) for string parsing
- FILESTREAM enabled for document storage (~80 GB)
- 6 linked server references to an Oracle system via OLE DB provider
- 42 SQL Agent jobs (T-SQL, PowerShell, and SSIS package steps)
- Service Broker queues for async order processing
- Distributed transactions (MSDTC) with a legacy accounting system
- BULK INSERT from a network file share (\\fileserver\imports\)
- Database Mail for alerting
- 4 logon triggers at the server level
- TDE enabled

Assess compatibility with BOTH:
1. Azure SQL Database (single database, General Purpose tier)
2. Azure SQL Managed Instance (General Purpose tier)

For each platform, classify each feature as:
- COMPATIBLE: Works as-is or with minimal config changes
- WORKAROUND: Requires code/architecture changes but is solvable
- BLOCKER: Not supported, requires removing or replacing the feature

Provide a recommendation on which platform is the better target.
```

## What the Agent Produced

The agent returned a structured comparison table covering both platforms. The broad strokes were correct:

**Azure SQL Database** had multiple blockers: cross-database queries (no support), linked servers (no support), SQL Agent (doesn't exist — would need Azure Data Factory or Elastic Jobs), Service Broker (not supported), FILESTREAM (not supported), distributed transactions (limited to elastic transactions only), CLR (not supported), and BULK INSERT from local/network paths (not supported — would need Azure Blob Storage).

**Azure SQL Managed Instance** was far more compatible: cross-database queries work within the same MI, CLR assemblies work at SAFE level, SQL Agent exists with most subsystems, Service Broker works within a single instance, Database Mail works, and TDE is supported. The blockers were fewer: FILESTREAM (not supported on MI either), linked servers to Oracle (supported but provider restrictions apply), distributed transactions with on-premises MSDTC (requires configuration and has limitations), and BULK INSERT from network paths (need Azure Blob Storage or a workaround).

The recommendation was Managed Instance with several remediation items.

## What I Validated and Changed

The agent's feature-level classification was mostly accurate, but several items needed correction or nuance:

- **Cross-database queries on MI.** The agent said "fully supported." That's true for databases *on the same Managed Instance*, but if the reference database stays on-premises or moves to a different MI, you're back to linked servers. I added a note that both databases must be on the same MI for cross-database queries to work without changes.

- **Distributed transactions.** The agent's assessment was slightly outdated. Azure SQL Managed Instance now supports distributed transactions *between Managed Instances* using managed DTC, but transactions with on-premises SQL Server via MSDTC still require careful networking setup (DTC endpoint exposure through Azure VNet). It's possible but not trivial. I reclassified this from "BLOCKER" to "WORKAROUND — requires networking and DTC configuration."

- **FILESTREAM.** The agent correctly flagged this as a blocker for both platforms. For the workaround, it suggested Azure Blob Storage with application-level changes, which is the right direction. I added that this is typically the single biggest refactoring item in FILESTREAM migrations — the application needs to change how it stores and retrieves documents.

- **SQL Agent job types.** The agent said MI supports SQL Agent, which is true, but glossed over the differences. SSIS package steps don't work the same way — you'd need Azure-SSIS Integration Runtime in Azure Data Factory. PowerShell steps work but with restricted cmdlet availability. CmdExec steps work but with no access to local file systems or network shares. Each of the 42 jobs needs individual assessment.

- **Logon triggers.** The agent didn't mention these at all. MI supports server-level DDL triggers and logon triggers, but behavior differs — particularly around Azure AD authentication flows. These need testing.

## The Compatibility Matrix

Here's the consolidated assessment after my validation:

| Feature | Azure SQL DB | Azure SQL MI | Notes |
|---|---|---|---|
| Cross-database queries | ❌ BLOCKER | ✅ COMPATIBLE | Both DBs must be on same MI |
| CLR assemblies (SAFE) | ❌ BLOCKER | ✅ COMPATIBLE | EXTERNAL_ACCESS needs config |
| FILESTREAM | ❌ BLOCKER | ❌ BLOCKER | Migrate to Azure Blob Storage |
| Linked servers (Oracle) | ❌ BLOCKER | ⚠️ WORKAROUND | Provider restrictions apply |
| SQL Agent jobs (T-SQL) | ❌ BLOCKER | ✅ COMPATIBLE | — |
| SQL Agent jobs (SSIS) | ❌ BLOCKER | ⚠️ WORKAROUND | Needs Azure-SSIS IR |
| SQL Agent jobs (PowerShell) | ❌ BLOCKER | ⚠️ WORKAROUND | Restricted cmdlets |
| Service Broker | ❌ BLOCKER | ✅ COMPATIBLE | Single instance only |
| Distributed transactions | ❌ BLOCKER | ⚠️ WORKAROUND | Managed DTC or VNet config |
| BULK INSERT (network path) | ❌ BLOCKER | ⚠️ WORKAROUND | Use Azure Blob Storage |
| Database Mail | ❌ BLOCKER | ✅ COMPATIBLE | — |
| TDE | ✅ COMPATIBLE | ✅ COMPATIBLE | — |
| Logon triggers | ❌ BLOCKER | ⚠️ WORKAROUND | Test with AAD auth flows |

**Verdict:** Azure SQL Database is not a viable target for this workload without major refactoring. Managed Instance is the better option, with FILESTREAM being the primary blocker requiring application changes.

## Complementary Tooling

The AI assessment is a first pass. Before committing to a migration, run these tools for deeper analysis:

- **Azure Migrate** with the Azure SQL assessment — discovers on-premises SQL instances, evaluates readiness, and provides SKU sizing recommendations.
- **Azure Data Migration Assistant (DMA)** — scans databases for compatibility issues specific to your target Azure SQL platform and version. It catches things at the engine level that text-based analysis misses.
- **Azure Database Migration Service** — handles the actual data movement, including online migrations with minimal downtime via continuous sync.

Use the AI agent's output as the high-level feasibility check, then validate with DMA for the detailed compatibility report. The agent is faster for the initial "should we even pursue this?" conversation with management. DMA is authoritative for the detailed engineering plan.

**One critical caveat:** Azure SQL capabilities change frequently. The agent's training data has a knowledge cutoff, and Microsoft ships Azure SQL updates weekly. Always verify compatibility claims against the [current T-SQL differences documentation on Microsoft Learn](https://learn.microsoft.com/en-us/azure/azure-sql/managed-instance/transact-sql-tsql-differences-sql-server). Treat the agent's output as a checklist to verify, not a definitive answer.

## Try This Yourself

Pick one production database and describe its feature usage to the agent — cross-database queries, CLR, linked servers, Agent jobs, any specialty features. Ask for a compatibility assessment against both Azure SQL Database and Managed Instance. Then compare the agent's output against Azure Data Migration Assistant's findings. You'll likely find the agent catches the architectural blockers faster (it takes two minutes versus DMA's full scan), while DMA catches engine-level nuances the agent misses (specific T-SQL syntax incompatibilities, unsupported hints, and collation issues).

For the full migration planning workflow — including deprecated feature scanning, compatibility level assessment, and runbook generation — see [Post 11: Migration Planning](/ai-for-dbas/migration-planning-compatibility/).

---

*Part of the [ALTER DBA ADD AGENT](/ai-for-dbas/alter-dba-add-agent/) series.*
