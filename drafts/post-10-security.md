# Security Audits: Finding What You Missed

Security audits are the thing every DBA knows they should do more often and rarely has time for. You check the obvious — sysadmin members, sa account status — and move on to the next fire. The gaps you don't check are the ones that show up in a penetration test report or, worse, a breach notification.

An AI agent doesn't make security audits exciting. But it makes them fast enough that you might actually do them regularly.

## Orphaned Users

Every environment has them. Employees leave, Active Directory groups change, databases get restored from backups — and user accounts accumulate like sediment.

```
Write a T-SQL script that finds orphaned users across all databases on this
instance. Check for:
1. Database users with no matching server login
2. Database users with no permissions and no role memberships (dead accounts)
3. SQL logins that are disabled at the server level but still mapped in databases
Group the results by database, include the user type (SQL, Windows, contained),
and flag any orphaned user that has elevated permissions (db_owner, db_securityadmin).
```

The agent generates a script that uses `sys.database_principals`, `sys.server_principals`, and `sys.database_role_members` to cross-reference across every database. The important detail: it flags orphaned users who still have elevated permissions — those are your highest-priority remediation targets.

**What this can't detect from SQL Server alone:** disabled Active Directory accounts (you need AD/Entra ID queries for that), and dormant contained database users (SQL Server doesn't track last authentication time without explicit auditing or Extended Events). Both are worth checking, but require tooling outside T-SQL.

**Try This Yourself:** Run the orphaned user query the agent generates. Most DBAs find at least a few surprises, especially on instances that have been running for years.

## Permission Sprawl: Who Has Sysadmin and Why?

This is the question every auditor asks, and too many DBAs answer by querying `sys.server_role_members` and calling it done. The real answer is more complex.

```
Write a comprehensive sysadmin membership audit that shows:
1. Direct members of the sysadmin role
2. Windows groups that are sysadmin members — and who is in those groups
3. Logins with CONTROL SERVER permission (effectively sysadmin without the role)
4. Service accounts and their permission level
5. For each entry, note whether it's a person, a service account, or a group
Flag anything that looks like it shouldn't be there: personal accounts with
sysadmin, groups with overly broad membership, or service accounts with
permissions beyond what they likely need.
```

The agent can't query Active Directory for group membership or classify accounts as "person" vs "service account" — that requires AD/Entra ID data. But it generates the SQL Server side: login-to-role mappings, CONTROL SERVER grants, and the audit structure that covers the paths auditors care about. You'll still need to expand Windows group membership and classify identities using AD tooling.

For a reusable, version-controlled approach to this kind of audit, I built [ai-security-audit](https://github.com/HannahVernon/ai-security-audit) — a set of AI-ready prompt templates for common security assessments.

## Connection Encryption

With TLS requirements tightening — especially for compliance frameworks like PCI-DSS and SOX — knowing which instances accept unencrypted connections matters.

```
Write a T-SQL query that checks connection encryption on this instance:
1. Query sys.dm_exec_connections to show how many active connections
   are encrypted vs unencrypted (encrypt_option column)
2. Check sys.dm_server_registry for certificate configuration if available
3. Show the protocol_type and net_transport for current connections
Note: Force Encryption is a configuration manager / registry setting,
not directly queryable from T-SQL. Flag that as a manual check.
```

The agent generates what T-SQL *can* tell you — current session encryption status — and correctly flags what it can't: Force Encryption configuration, TLS protocol version negotiation (the DMVs show TDS protocol, not TLS version), and certificate binding details. For those, you need server-level inspection or external tooling.

For a fleet-wide TLS audit that covers the full picture — certificate details, SPN diagnostics, protocol configuration, and health checks across all your instances — [sql-cert-inspector](https://github.com/HannahVernon/sql-cert-inspector) handles the parts T-SQL can't reach. We covered [automating it with PowerShell](/ai-for-dbas/automating-server-health-checks-and-inventory-scripts/) in an earlier post.

## Dynamic SQL Injection Surface

This is the audit most DBAs never do — and it's the one that matters most from a security perspective. How many of your stored procedures build dynamic SQL by concatenating user input?

```
Scan all stored procedure definitions in this database (from sys.sql_modules)
and identify procedures that:
1. Use string concatenation to build dynamic SQL (EXEC(@sql) or sp_executesql
   with concatenated parameters)
2. Accept varchar/nvarchar parameters that flow into the dynamic SQL
3. Do NOT use sp_executesql with proper parameterization
4. Do NOT use QUOTENAME for object name injection protection
Rank by risk: procedures that concatenate user-supplied values directly into
executable SQL are highest risk.
```

The agent parses procedure definitions and flags the risky patterns. This is a **triage scan**, not a definitive security audit — it's heuristic pattern matching on source text with significant blind spots: encrypted modules are invisible, CLR procedures bypass T-SQL entirely, SQL Agent job steps contain their own dynamic SQL, and complex variable flow through multiple assignments can hide concatenation from simple text analysis. And note that `QUOTENAME` only protects against **identifier injection** (table/column names), not arbitrary value injection — those still need parameterized `sp_executesql`.

But as a first pass to find the procedures worth deeper investigation, it beats manually reading through hundreds of module definitions. In my experience, legacy codebases almost always have a few procedures that concatenate parameters directly into `EXEC()` calls with no sanitization.

This is exactly the kind of audit that becomes a [satellite post](/ai-for-dbas/alter-dba-add-agent/) with a deeper dive into remediation patterns.

## Compliance Report Generation

Auditors want evidence. They want it formatted their way. And they want it now.

```
Generate a PCI-DSS v4.0 evidence collection script for the SQL Server
database engine scope. For each control, write a diagnostic query that
collects what SQL Server metadata can prove, and note what requires
evidence from outside SQL Server (OS, AD, network, process documentation).
Focus on:
- Requirement 2.2.7: System components configured to prevent misuse
- Requirement 7.2: Access limited to need-to-know
- Requirement 8.2.2: No shared/generic accounts
- Requirement 10.2: Audit trail configuration
```

The agent generates queries for what SQL Server metadata can actually evidence — login configurations, role memberships, audit settings — and flags what it can't prove from T-SQL alone. And that's a long list: "no default passwords" requires password policy verification outside the engine, "unnecessary services disabled" is an OS/configuration concern, and "audit trail for cardholder data access" depends entirely on your scope definition and whether you've already configured SQL Server Audit or Extended Events to capture those events.

Use the generated scripts as a **starting point** for evidence collection, not as a finished compliance package. The agent's knowledge of compliance frameworks comes from training data, not from a certified QSA. Your auditor has the final word on what constitutes sufficient evidence — and they will have opinions about your scope, your compensating controls, and your evidence format that no AI can anticipate.

## Reviewing Your Own Security Scripts

One of the most useful patterns: point the agent at your *existing* security scripts and ask it to find the gaps.

```
Review the security audit scripts in the audit/ folder. For each script, tell me:
1. What it checks
2. What it misses — common attack vectors or permission paths it doesn't cover
3. Whether it handles contained databases, Azure AD / Entra ID users,
   and cross-database ownership chains
4. Any false positives or false negatives you can identify
```

The agent reviews your audit scripts with the same rigor you'd apply to production code. It often catches edge cases: contained database users that bypass server-level audits, Windows group nesting that hides effective permissions, cross-database ownership chains that grant unintended access.

## The Audit Areas You Probably Aren't Checking

Beyond the sections above, here are security surface areas that a thorough audit should cover — and that the agent can help you script:

- **`guest` user enabled** in user databases (it shouldn't be, except in `msdb`)
- **`public` role grants** beyond defaults — one of the most common permission sprawl vectors
- **TRUSTWORTHY** database property enabled (escalation path to sysadmin)
- **Cross-database ownership chaining** enabled at instance level
- **Database owners** set to `sa` or high-privilege accounts when they shouldn't be
- **Linked servers** with saved credentials or overly permissive security contexts
- **SQL Agent job ownership** and proxy/credential configuration
- **xp_cmdshell, OLE Automation, Ad Hoc Distributed Queries, CLR** — surface area configuration
- **Server-level impersonation grants** that create privilege escalation paths
- **SQL Server Audit** — is it actually configured and running, or just planned?

Each of these is a prompt away from a diagnostic script. The agent won't know your environment's risk tolerance — that's your judgment call — but it can enumerate the current state faster than you can type the DMV names.

---

Future satellite posts will dive deeper into specific security topics— [orphaned user remediation](/ai-for-dbas/alter-dba-add-agent/), [SQL injection surface scanning](/ai-for-dbas/alter-dba-add-agent/), [TDE certificate management](/ai-for-dbas/alter-dba-add-agent/), and more.

**Next up:** [Migration Planning: Compatibility Checks and Deprecated Features](/ai-for-dbas/migration-planning-compatibility-checks-and-deprecated-features/) — using AI to plan and de-risk SQL Server version upgrades.


---

*Part of the [ALTER DBA ADD AGENT](/ai-for-dbas/alter-dba-add-agent/) series — [Previous: Incident Response](/ai-for-dbas/incident-response-root-cause-analysis-with-an-ai-partner/)*
