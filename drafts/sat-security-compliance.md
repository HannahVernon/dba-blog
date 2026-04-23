Audit season is the worst time to be a DBA. An auditor sends a spreadsheet of controls, each one requiring specific evidence from your SQL Server environment, and you have two weeks to produce it all. You spend those two weeks writing one-off queries, taking screenshots, reformatting output into the template your compliance team requires, and wondering if you missed something.

AI agents don't replace your compliance team or your QSA. But they dramatically reduce the time between "here's the control requirement" and "here's the evidence query." Instead of translating compliance language into T-SQL from scratch, you describe the requirement and get a first-draft query in seconds. I covered this at a high level in [Security Audits: Finding What You Missed](/ai-for-dbas/security-audits-finding-missed/). This post walks through specific examples for SOX, PCI-DSS, and HIPAA.

## The Problem: Compliance Language ≠ SQL Server Language

Auditors speak in control requirements: "Demonstrate that access to cardholder data is restricted to authorized personnel." You speak in DMVs: `sys.database_permissions`, `sys.server_role_members`, `sys.dm_exec_sessions`. The translation between the two is where all the time goes — not the query itself, but figuring out which metadata answers the auditor's question.

This is where the AI agent earns its keep. It understands both languages.

## Example 1: PCI-DSS — Encryption at Rest (TDE Audit)

PCI-DSS v4.0 Requirement 3.5.1 requires that stored account data is secured with strong cryptography. For SQL Server, that typically means TDE (Transparent Data Encryption) on databases containing cardholder data.

```text
Generate a T-SQL evidence collection script for PCI-DSS Requirement 3.5.1.
I need to prove that databases containing cardholder data are encrypted
with TDE. Show:
1. TDE status for every database on the instance
2. Certificate details — name, expiration date, key length
3. Whether the certificate is backed up (check msdb backup history)
4. DEK (Database Encryption Key) algorithm and key length
5. Encryption scan state (in progress, complete, etc.)
Flag any database that is NOT encrypted and note it as requiring
manual justification for why TDE is not enabled.
```

The agent produced a clean script joining `sys.databases`, `sys.dm_database_encryption_keys`, and `sys.certificates`:

```sql
/* PCI-DSS 3.5.1 Evidence — TDE Status Audit */
SET NOCOUNT ON;

SELECT
    d.[name]                                AS [DatabaseName]
   ,CASE
        WHEN dek.[encryption_state] IS NULL THEN N'NOT ENCRYPTED'
        WHEN dek.[encryption_state] = 0     THEN N'No DEK — not encrypted'
        WHEN dek.[encryption_state] = 1     THEN N'Unencrypted'
        WHEN dek.[encryption_state] = 2     THEN N'Encryption in progress'
        WHEN dek.[encryption_state] = 3     THEN N'Encrypted'
        WHEN dek.[encryption_state] = 4     THEN N'Key change in progress'
        WHEN dek.[encryption_state] = 5     THEN N'Decryption in progress'
        WHEN dek.[encryption_state] = 6     THEN N'Protection change in progress'
        ELSE CONVERT(nvarchar(20), dek.[encryption_state])
    END                                     AS [EncryptionStatus]
   ,c.[name]                                AS [CertificateName]
   ,c.[expiry_date]                         AS [CertificateExpiry]
   ,CASE
        WHEN c.[expiry_date] < GETDATE()
            THEN N'EXPIRED'
        WHEN c.[expiry_date] < DATEADD(DAY, 90, GETDATE())
            THEN N'EXPIRING SOON'
        ELSE N'OK'
    END                                     AS [CertificateStatus]
   ,c.[key_length]                          AS [CertificateKeyLength]
   ,dek.[key_algorithm]                     AS [DEKAlgorithm]
   ,dek.[key_length]                        AS [DEKKeyLength]
   ,dek.[percent_complete]                  AS [EncryptionProgress]
   ,CASE
        WHEN dek.[encryption_state] IS NULL THEN NULL
        WHEN dek.[encryption_state] = 1     THEN N'Unencrypted'
        WHEN dek.[encryption_state] = 2     THEN N'Encryption in progress'
        WHEN dek.[encryption_state] = 3     THEN N'Encrypted'
        WHEN dek.[encryption_state] = 4     THEN N'Key change in progress'
        WHEN dek.[encryption_state] = 5     THEN N'Decryption in progress'
        WHEN dek.[encryption_state] = 6     THEN N'Protection change in progress'
        ELSE CONVERT(nvarchar(50), dek.[encryption_state])
    END                                     AS [EncryptionStateDesc]
   ,N'MANUAL VERIFICATION REQUIRED — certificate backups use BACKUP CERTIFICATE '
    + N'which does not log to msdb.dbo.backupset. Check your key management '
    + N'logs or backup file system for .cer and .pvk files.'
                                            AS [CertificateBackupStatus]
FROM sys.[databases] AS d
    LEFT JOIN sys.[dm_database_encryption_keys] AS dek
        ON d.[database_id] = dek.[database_id]
    LEFT JOIN [master].sys.[certificates] AS c
        ON dek.[encryptor_thumbprint] = c.[thumbprint]
WHERE d.[database_id] > 4  /* exclude system databases */
ORDER BY
    CASE WHEN dek.[encryption_state] = 3 THEN 1 ELSE 0 END
   ,d.[name];
```

What I changed: the certificate backup check was too generic — it was just looking for any `master` database backup, not specifically a certificate backup. SQL Server doesn't track certificate backups in `msdb.backupset` directly (certificate backups go through `BACKUP CERTIFICATE`, which doesn't log to `backupset`). I replaced the `OUTER APPLY` with a note that certificate backup verification requires checking your backup file system or your certificate backup automation logs. This is important to flag honestly — telling an auditor you have a backup when you're really checking something else is worse than saying "manual verification required."

## Example 2: SOX — Access Control Evidence

SOX (Sarbanes-Oxley) controls around IT general controls require evidence that access to financial systems is appropriately restricted and reviewed. The specific evidence varies by auditor, but you'll almost always need to show login inventory and privilege assignments.

```text
Generate a SOX access control evidence script for SQL Server. Show:
1. All server logins — name, type (SQL, Windows, Entra ID), create date,
   last modification date, disabled status
2. Server-level role memberships for each login
3. Logins with sysadmin access (direct or through Windows groups)
4. SQL logins with password policy enforcement disabled
5. Logins created or modified in the last 90 days (change evidence)
Format the output so it can be pasted into a compliance spreadsheet.
```

```sql
/* SOX Access Control Evidence — Login Inventory */
SET NOCOUNT ON;

SELECT
    sp.[name]                               AS [LoginName]
   ,sp.[type_desc]                          AS [LoginType]
   ,sp.[create_date]                        AS [CreatedDate]
   ,sp.[modify_date]                        AS [LastModified]
   ,CASE sp.[is_disabled]
        WHEN 1 THEN N'DISABLED'
        ELSE N'Active'
    END                                     AS [Status]
   ,COALESCE(roles.[ServerRoles], N'None')  AS [ServerRoles]
   ,CASE
        WHEN roles.[ServerRoles] LIKE N'%sysadmin%'
            THEN N'YES — review required'
        ELSE N'No'
    END                                     AS [HasSysadmin]
   ,CASE
        WHEN sp.[type] = 'S' AND sl.[is_policy_checked] = 0
            THEN N'POLICY NOT ENFORCED'
        WHEN sp.[type] = 'S' AND sl.[is_expiration_checked] = 0
            THEN N'EXPIRATION NOT ENFORCED'
        WHEN sp.[type] = 'S'
            THEN N'Policy enforced'
        ELSE N'N/A (Windows/Entra)'
    END                                     AS [PasswordPolicy]
   ,CASE
        WHEN sp.[create_date] > DATEADD(DAY, -90, GETDATE())
            THEN N'NEW — created within 90 days'
        WHEN sp.[modify_date] > DATEADD(DAY, -90, GETDATE())
            THEN N'MODIFIED within 90 days'
        ELSE N''
    END                                     AS [RecentChangeFlag]
   ,CASE
        WHEN sp.[name] LIKE N'NT SERVICE%'
            OR sp.[name] LIKE N'NT AUTHORITY%'
            THEN CONVERT(bit, 1)
        ELSE CONVERT(bit, 0)
    END                                     AS [IsServiceAccount]
FROM sys.[server_principals] AS sp
    LEFT JOIN sys.[sql_logins] AS sl
        ON sp.[principal_id] = sl.[principal_id]
    OUTER APPLY (
        SELECT STRING_AGG(sr.[name], N', ') AS [ServerRoles]
        FROM sys.[server_role_members] AS srm
            INNER JOIN sys.[server_principals] AS sr
                ON srm.[role_principal_id] = sr.[principal_id]
        WHERE srm.[member_principal_id] = sp.[principal_id]
    ) AS roles
WHERE sp.[type] IN ('S', 'U', 'G', 'E', 'X')
    AND sp.[name] NOT LIKE N'##%'        /* exclude certificates */
ORDER BY
    CASE
        WHEN roles.[ServerRoles] LIKE N'%sysadmin%' THEN 0
        ELSE 1
    END
   ,sp.[name];
```

The agent's initial version filtered out service accounts entirely. I kept them in with a flag — auditors want to see *everything* and then verify that service accounts are appropriately justified. Hiding accounts from an audit report is a bad habit.

## Example 3: HIPAA — Audit Trail Configuration

HIPAA's Security Rule (§164.312(b)) requires audit controls for information systems containing electronic protected health information (ePHI). For SQL Server, this means verifying that auditing is configured and operational.

```text
Generate a HIPAA audit trail evidence script. Show:
1. Whether SQL Server Audit is configured and active
2. The audit specifications — what events are being captured
3. Audit log destination and current status
4. Whether any audit has failed or is in an error state
5. If C2 audit mode is enabled (legacy but still used)
6. Extended Events sessions that capture security-relevant events
```

```sql
/* HIPAA Audit Trail Evidence — Audit Configuration Status */
SET NOCOUNT ON;

/* Server-level audit configuration */
SELECT
    sa.[name]                               AS [AuditName]
   ,sa.[status_desc]                        AS [AuditStatus]
   ,sa.[type_desc]                          AS [AuditDestination]
   ,sa.[log_file_path]                      AS [LogFilePath]
   ,sa.[log_file_size]                      AS [LogFileSizeMB]
   ,sa.[max_rollover_files]                 AS [MaxRolloverFiles]
   ,sa.[on_failure_desc]                    AS [OnFailureAction]
   ,sa.[create_date]                        AS [CreatedDate]
   ,sa.[modify_date]                        AS [LastModified]
FROM sys.[server_audits] AS sa
ORDER BY sa.[name];

/* Server audit specifications — what's being captured */
SELECT
    sa.[name]                               AS [AuditName]
   ,sas.[name]                              AS [SpecificationName]
   ,CASE sas.[is_state_enabled]
        WHEN 1 THEN N'ENABLED'
        ELSE N'DISABLED — NOT CAPTURING'
    END                                     AS [SpecificationStatus]
   ,sasd.[audit_action_name]                AS [AuditedAction]
   ,sasd.[class_desc]                       AS [ObjectClass]
   ,sasd.[audited_result]                   AS [AuditedResult]
FROM sys.[server_audit_specifications] AS sas
    INNER JOIN sys.[server_audits] AS sa
        ON sas.[audit_guid] = sa.[audit_guid]
    INNER JOIN sys.[server_audit_specification_details] AS sasd
        ON sas.[server_specification_id] = sasd.[server_specification_id]
ORDER BY sa.[name], sasd.[audit_action_name];

/* Database-level audit specifications */
SELECT
    DB_NAME()                               AS [DatabaseName]
   ,sa.[name]                               AS [AuditName]
   ,das.[name]                              AS [SpecificationName]
   ,CASE das.[is_state_enabled]
        WHEN 1 THEN N'ENABLED'
        ELSE N'DISABLED — NOT CAPTURING'
    END                                     AS [SpecificationStatus]
   ,dasd.[audit_action_name]                AS [AuditedAction]
   ,dasd.[class_desc]                       AS [ObjectClass]
   ,dasd.[audited_result]                   AS [AuditedResult]
FROM sys.[database_audit_specifications] AS das
    INNER JOIN sys.[server_audits] AS sa
        ON das.[audit_guid] = sa.[audit_guid]
    INNER JOIN sys.[database_audit_specification_details] AS dasd
        ON das.[database_specification_id]
           = dasd.[database_specification_id]
ORDER BY sa.[name], dasd.[audit_action_name];

/* Legacy C2 audit mode check */
SELECT
    [name]                                  AS [ConfigOption]
   ,[value]                                 AS [ConfiguredValue]
   ,[value_in_use]                          AS [RunningValue]
   ,CASE CONVERT(int, [value_in_use])
        WHEN 1 THEN N'ENABLED (legacy — consider migrating to SQL Server Audit)'
        ELSE N'Disabled'
    END                                     AS [Status]
FROM sys.[configurations]
WHERE [name] = N'c2 audit mode';

/* Extended Events sessions capturing security events */
SELECT
    es.[name]                               AS [SessionName]
   ,CASE es.[startup_state]
        WHEN 1 THEN N'Auto-start'
        ELSE N'Manual'
    END                                     AS [StartupBehavior]
   ,CASE
        WHEN r.[name] IS NOT NULL THEN N'RUNNING'
        ELSE N'STOPPED'
    END                                     AS [CurrentStatus]
   ,COALESCE(evt.[EventList], N'(query session XML for details)')
                                            AS [CapturedEvents]
FROM sys.[server_event_sessions] AS es
    LEFT JOIN sys.[dm_xe_sessions] AS r
        ON es.[name] = r.[name]
    OUTER APPLY (
        SELECT STRING_AGG(se.[name], N', ') AS [EventList]
        FROM sys.[server_event_session_events] AS se
        WHERE se.[event_session_id] = es.[event_session_id]
            AND se.[name] IN (
                N'audit_event'
               ,N'error_reported'
               ,N'login'
               ,N'logout'
               ,N'sql_statement_completed'
               ,N'rpc_completed'
            )
    ) AS evt
WHERE es.[name] NOT LIKE N'system_health'
ORDER BY es.[name];
```

## What I Validated Across All Three

A few things I checked before using any of these in a real audit:

1. **The queries don't modify anything.** All reads, no writes. An auditor should be able to run these without concern — and you should be able to prove they're read-only.

2. **The output format works for compliance teams.** I copy-paste the results into Excel. The column names are descriptive enough that a non-DBA can read them. If your compliance team has a specific template, ask the agent to match it — provide the column headers and it'll restructure the query.

3. **The agent's compliance knowledge has limits.** It knows the general structure of PCI-DSS, SOX, and HIPAA requirements from training data. It does *not* know your specific scope, your compensating controls, or your auditor's interpretation of a requirement. Always have your compliance team or QSA review the evidence before submission.

## Try This Yourself

Pick one compliance control you need to evidence next audit cycle. Describe the requirement to the agent in plain English — include the framework name, the specific control number, and what the auditor is looking for. Then compare the agent's output against what you've been producing manually.

The time savings compound. Once you have a working evidence query for each control, save them in a version-controlled repository (we covered that in [Version Control for DBAs](/ai-for-dbas/version-control-cicd/)). Next audit cycle, you run the scripts and update the evidence — instead of rebuilding everything from memory.

The goal isn't to automate compliance. It's to automate the *evidence gathering* so you can spend your time on what actually matters: understanding and improving your security posture. The agent translates compliance language into T-SQL. You verify the results are accurate and the evidence is honest.

For the full security audit workflow, see [Security Audits: Finding What You Missed](/ai-for-dbas/security-audits-finding-missed/).

---

*Part of the [ALTER DBA ADD AGENT](/ai-for-dbas/alter-dba-add-agent/) series.*
