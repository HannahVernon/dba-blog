# ALTER DBA ADD AGENT: Practical AI for Database Professionals

If you're a SQL Server DBA who's heard the buzz about AI but isn't sure what it means for your daily work, this series is for you. No marketing fluff, no hand-waving — just a working DBA showing what happens when you point an AI coding agent at real database problems.

Each post covers a specific task area with real prompts, real output, and real tradeoffs. Start anywhere that matches what's on your plate today.

## The Series

### Getting Oriented

1. **[The DBA's Blind Spot: Why AI Coding Agents Are Coming for Your Workflow](/ai-for-dbas/the-dbas-blind-spot/)** — Code completion vs. coding agents, and why the difference matters more than you think.

2. **[What Can an AI Coding Agent Actually Do for a DBA?](/ai-for-dbas/what-can-ai-agent-do-for-dba/)** — A tour of every task area where these tools add value, from T-SQL to security audits.

3. **[Getting Started: Your First Hour with GitHub Copilot CLI](/ai-for-dbas/getting-started-copilot-cli/)** — Installation, first prompts, and what to expect when you fire it up.

### Daily Work

4. **[Writing T-SQL with an AI Partner](/ai-for-dbas/writing-tsql-with-ai-partner/)** — Generating, reviewing, and refactoring T-SQL with an agent that reads your schema.
   - [AI-Assisted Implicit Join Conversion](/ai-for-dbas/ai-convert-implicit-joins/) — Converting legacy implicit joins to ANSI syntax
   - [AI-Assisted Window Function Development](/ai-for-dbas/ai-window-functions/) — Building and debugging window functions
   - [AI-Assisted Cursor-to-Set-Based Conversion](/ai-for-dbas/ai-cursor-to-set-based/) — Replacing cursors with set-based operations
   - [AI-Assisted Parameter Sniffing Diagnosis](/ai-for-dbas/ai-parameter-sniffing-diagnosis/) — Diagnosing and fixing parameter sniffing issues

5. **[Automating Server Health Checks and Inventory Scripts](/ai-for-dbas/health-checks-inventory/)** — Building the scripts that keep your fleet healthy, faster than you'd write them alone.
   - [AI-Assisted Backup Status Audit](/ai-for-dbas/ai-backup-status-audit/) — Auditing backup status across your fleet
   - [AI-Assisted Version and Patch Inventory](/ai-for-dbas/ai-version-patch-inventory/) — Building version inventories
   - [AI-Assisted AG Topology Mapping](/ai-for-dbas/ai-ag-topology-mapping/) — Documenting AG configurations
   - [AI-Assisted Compatibility Level Audit](/ai-for-dbas/ai-compat-level-audit/) — Finding compat level mismatches

6. **[PowerShell Automation: Backups, Maintenance, and AG Management](/ai-for-dbas/powershell-automation/)** — AG failover scripts, backup validation, and maintenance automation with AI assistance.
   - [AI-Assisted Backup Automation with PowerShell](/ai-for-dbas/ai-backup-automation-powershell/) — Building robust backup scripts
   - [AI-Assisted AG Failover Runbook](/ai-for-dbas/ai-ag-failover-runbook/) — AG failover procedures
   - [AI-Assisted DBCC CHECKDB Scheduling](/ai-for-dbas/ai-dbcc-checkdb-scheduling/) — Scheduling integrity checks
   - [AI-Assisted Patching Pre/Post Checks](/ai-for-dbas/ai-patching-pre-post-checks/) — Patch readiness automation

7. **[Understanding Unfamiliar Code: Reverse-Engineering Legacy Procedures](/ai-for-dbas/reverse-engineering-legacy-procedures/)** — That 2,000-line stored procedure nobody wants to touch? The agent will read it.
   - [AI-Assisted Dynamic SQL Dissection](/ai-for-dbas/ai-dynamic-sql-dissection/) — Understanding complex dynamic SQL
   - [AI-Assisted Dependency Mapping](/ai-for-dbas/ai-dependency-mapping/) — Mapping object dependencies
   - [AI-Assisted Cursor Elimination](/ai-for-dbas/ai-cursor-elimination/) — Step-by-step cursor replacement

### Troubleshooting

8. **[Wait Stats, Deadlocks, and Blocking Chains: AI-Assisted Diagnosis](/ai-for-dbas/wait-stats-deadlocks-blocking/)** — Turning raw wait stats and deadlock graphs into actionable explanations.
   - [AI-Assisted CXPACKET Diagnosis](/ai-for-dbas/ai-cxpacket-diagnosis/) — Diagnosing parallelism waits
   - [AI-Assisted Deadlock Analysis](/ai-for-dbas/ai-deadlock-analysis/) — Reading deadlock graphs
   - [AI-Assisted Blocking Chain Triage](/ai-for-dbas/ai-blocking-chain-triage/) — Resolving blocking chains
   - [AI-Assisted Memory Grant Analysis](/ai-for-dbas/ai-memory-grant-analysis/) — Diagnosing memory grant issues

9. **[Incident Response: Root Cause Analysis with an AI Partner](/ai-for-dbas/incident-response-root-cause/)** — When the pager goes off at 2 AM, having a tireless partner changes the equation.

### Security and Compliance

10. **[Security Audits: Finding What You Missed](/ai-for-dbas/security-audits-finding-missed/)** — Orphaned users, permission sprawl, and the gaps that accumulate over years.
    - [AI-Assisted Orphaned User Cleanup](/ai-for-dbas/ai-orphaned-users/) — Finding and fixing orphaned users
    - [AI-Assisted Permission Sprawl Analysis](/ai-for-dbas/ai-permission-sprawl/) — Auditing permission creep
    - [AI-Assisted SQL Injection Scanning](/ai-for-dbas/ai-sql-injection-scan/) — Finding injection vulnerabilities
    - [AI-Assisted Compliance Checklists](/ai-for-dbas/ai-compliance-checklists/) — SOX/PCI/HIPAA audit automation

### Migration and Modernization

11. **[Migration Planning: Compatibility Checks and Deprecated Features](/ai-for-dbas/migration-planning-compatibility/)** — Scanning for deprecated syntax, compatibility issues, and migration blockers.
    - [AI-Assisted Deprecated Feature Scanning](/ai-for-dbas/ai-deprecated-feature-scan/) — Finding deprecated features
    - [AI-Assisted Azure SQL Feasibility Assessment](/ai-for-dbas/ai-azure-sql-feasibility/) — Evaluating Azure migration readiness
    - [AI-Assisted Post-Migration Validation](/ai-for-dbas/ai-post-migration-validation/) — Verifying migration success

### Monitoring

12. **[AI-Native Monitoring: PerformanceMonitor, PerformanceStudio, and the MCP Revolution](/ai-for-dbas/ai-native-monitoring/)** — Tools built from the ground up to work with AI agents, featuring Erik Darling's PerformanceMonitor and PerformanceStudio.

13. **[Building Custom Monitoring Queries and Alerts](/ai-for-dbas/building-custom-monitoring/)** — When off-the-shelf monitoring doesn't fit, build exactly what you need.
    - [AI-Assisted Long-Running Transaction Alerts](/ai-for-dbas/ai-long-running-transaction-alerts/) — Monitoring open transactions
    - [AI-Assisted Blocking Alerts](/ai-for-dbas/ai-blocking-alerts/) — Building blocking chain alerts
    - [AI-Assisted AG Health Monitoring](/ai-for-dbas/ai-ag-health-monitoring/) — Availability Group health checks
    - [AI-Assisted Backup Failure Alerts](/ai-for-dbas/ai-backup-failure-alerts/) — Backup monitoring automation

### Infrastructure and Process

14. **[Version Control and CI/CD: Unlocking What the Agent Can Actually Do](/ai-for-dbas/version-control-cicd/)** — AI agents get dramatically more useful when your database code is in a repo.

15. **[Teaching AI Your Environment: Custom Instructions and Context](/ai-for-dbas/custom-instructions-context/)** — Making the agent deeply effective by teaching it your standards, naming conventions, and environment.

16. **[AI-Assisted Pull Request Reviews for Database Code](/ai-for-dbas/ai-assisted-pr-reviews/)** — Schema changes, deployment scripts, and permission audits — reviewed before the senior DBA even looks.

### The Bigger Picture

17. **[The AI-Augmented DBA Team: Mentoring and Knowledge Transfer](/ai-for-dbas/ai-augmented-dba-team/)** — What changes when the whole team has AI partners, not just one person.

18. **[How This Series Was Written: A DBA and an AI Walk Into a Terminal](/ai-for-dbas/how-this-series-was-written/)** — The meta post. Every word in this series was co-authored with the tools it describes. Here's how.

### For Junior DBAs

- **[A Junior DBA's Field Guide to This Series](/ai-for-dbas/junior-dba-field-guide/)** — New to the profession? This companion guide maps the vocabulary walls, production safety gaps, and confidence calibration challenges you'll hit. Senior DBAs: share this with your juniors before pointing them at the series.

---

## Source Materials

The complete working directory for this series — every draft, editorial decision, and iteration — is available on GitHub:

**[github.com/HannahVernon/dba-blog](https://github.com/HannahVernon/dba-blog)**
