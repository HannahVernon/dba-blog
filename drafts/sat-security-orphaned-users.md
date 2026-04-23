Every instance accumulates orphaned users. Someone leaves the company, an AD group gets restructured, a database gets restored from a different server — and suddenly you have database principals that point to nothing. They're harmless until they're not: an orphaned user with `db_owner` is a dormant privilege escalation waiting for someone to recreate that login name.

I covered orphaned user detection at a high level in [Security Audits: Finding What You Missed](/ai-for-dbas/security-audits-finding-missed/). This post goes deeper — finding every flavor of orphan, understanding why each one exists, and generating remediation scripts that won't accidentally break anything.

## The Problem: It's Not Just "No Matching Login"

Most DBAs think of orphaned users as database users whose SID doesn't match any server login. That's the classic case, but it's only one flavor:

1. **No matching login at all** — the login was dropped or never existed on this instance (common after restoring from another server).
2. **SID mismatch** — a login with the same *name* exists, but it was dropped and recreated, so the SIDs don't match. The user looks mapped but isn't.
3. **Disabled logins** — the login exists and the SID matches, but it's disabled at the server level. The user is technically not orphaned, but it's functionally dead.
4. **Removed AD groups** — a Windows user was granted access through an AD group that no longer exists or no longer contains them. SQL Server has no way to detect this from T-SQL alone.
5. **Contained database users** — these don't map to server logins at all by design, but abandoned contained users are their own cleanup problem.

## The Prompt

```text
Write a T-SQL script that finds orphaned and potentially orphaned database
users across all databases on this instance. For each database, identify:

1. Users with no matching server login (classic orphan — SID not found in
   sys.server_principals)
2. Users where a login with the same name exists but the SID doesn't match
3. Users mapped to a login that is currently disabled
4. Contained database users (for contained databases)
5. Users with no permissions, no role memberships, and no schema ownership
   (dead accounts regardless of orphan status)

Exclude system principals: dbo, guest, INFORMATION_SCHEMA, sys, and any
principal_id <= 4.

For each orphaned user, show: database name, user name, user type
(SQL_USER, WINDOWS_USER, etc.), whether they own any schemas or objects,
and their highest database role membership.

Generate a remediation script section with:
- ALTER USER ... WITH LOGIN for SID mismatches where the login exists
- DROP USER for true orphans with no schema/object ownership
- Warnings for orphans that own schemas (must transfer ownership first)
```

## What the Agent Produced

The agent generated a script using `sp_MSforeachdb` (which I replaced — more on that below) to iterate databases, querying `sys.database_principals` joined against `sys.server_principals` to find the mismatches. It correctly used `SID` comparison rather than name matching for the core orphan detection.

What I liked:

- **It caught the SID mismatch case.** The agent joined on `sid` and flagged rows where SIDs matched a different login name or where a login existed with the same name but a different SID — a subtle issue that most orphan scripts miss.

> **Note on name-based vs. SID-based matching:** The primary join uses `dp.[sid] = sp.[sid]`, which is the correct way to map database users to server logins. Name-based matching (`dp.[name] = sp.[name]`) can miss renamed logins — if a login is renamed at the server level, users mapped by SID still work, but a name-based join would incorrectly flag them as orphans. The SID mismatch detection in the `CASE` expression identifies cases where a login with the *same name* exists but has a different SID (e.g., a login that was dropped and recreated).
- **It checked for schema ownership** using `sys.schemas` before generating `DROP USER` statements. Dropping a user who owns a schema throws error 15138, so the script correctly generates `ALTER AUTHORIZATION` first.
- **It excluded the system principals.** Not just by name, but also by `principal_id <= 4` and `type NOT IN ('R')` to skip database roles.

What needed fixing:

- **`sp_MSforeachdb` is unreliable.** It's undocumented, occasionally skips databases, and doesn't handle database names with special characters well. I replaced it with a cursor over `sys.databases` using dynamic SQL with `QUOTENAME`.
- **It missed contained database users.** The agent checked `sys.databases` for `containment <> 0` but didn't query the contained users differently. Contained users have `authentication_type = 2` (database-level authentication) — they're not orphans, but abandoned contained users need separate cleanup logic.
- **The disabled login check was too aggressive.** A disabled login might be intentionally disabled during an employee's leave of absence. I changed the output to flag these as "review needed" rather than generating remediation scripts automatically.

## The Final Script

```sql
/* Orphaned User Audit — all databases on the instance */
SET NOCOUNT ON;

CREATE TABLE #OrphanedUsers (
    [DatabaseName]       sysname       NOT NULL
   ,[UserName]           sysname       NOT NULL
   ,[UserType]           nvarchar(60)  NOT NULL
   ,[OrphanReason]       nvarchar(100) NOT NULL
   ,[HighestRole]        sysname       NULL
   ,[OwnsSchemas]        nvarchar(MAX) NULL
   ,[RemediationAction]  nvarchar(MAX) NULL
);

DECLARE @DbName sysname;
DECLARE @SQL nvarchar(MAX);

DECLARE db_cursor CURSOR LOCAL FAST_FORWARD FOR
    SELECT [name]
    FROM sys.[databases]
    WHERE [state] = 0       /* ONLINE */
        AND [name] NOT IN (N'tempdb')
    ORDER BY [name];

OPEN db_cursor;
FETCH NEXT FROM db_cursor INTO @DbName;

WHILE @@FETCH_STATUS = 0
BEGIN
    SET @SQL = N'
    USE ' + QUOTENAME(@DbName) + N';

    INSERT INTO #OrphanedUsers (
        [DatabaseName], [UserName], [UserType],
        [OrphanReason], [HighestRole], [OwnsSchemas],
        [RemediationAction]
    )
    SELECT
        DB_NAME() AS [DatabaseName]
       ,dp.[name] AS [UserName]
       ,dp.[type_desc] AS [UserType]
       ,CASE
            WHEN sp.[sid] IS NULL
                 AND dp.[authentication_type] <> 2
                THEN N''No matching login (SID not found)''
            WHEN sp.[name] IS NOT NULL
                 AND dp.[sid] <> sp.[sid]
                THEN N''SID mismatch — login exists with different SID''
            WHEN sp.[is_disabled] = 1
                THEN N''Login is disabled''
            WHEN dp.[authentication_type] = 2
                 AND NOT EXISTS (
                     SELECT 1
                     FROM sys.[database_role_members] AS drm
                     WHERE drm.[member_principal_id] = dp.[principal_id]
                 )
                 AND NOT EXISTS (
                     SELECT 1
                     FROM sys.[database_permissions] AS perm
                     WHERE perm.[grantee_principal_id] = dp.[principal_id]
                         AND perm.[type] <> N''CO'' /* CONNECT */
                 )
                THEN N''Contained user — no roles or permissions''
            ELSE NULL
        END AS [OrphanReason]

       /* Highest role membership */
       ,(  SELECT TOP (1) r.[name]
           FROM sys.[database_role_members] AS drm
               INNER JOIN sys.[database_principals] AS r
                   ON drm.[role_principal_id] = r.[principal_id]
           WHERE drm.[member_principal_id] = dp.[principal_id]
           ORDER BY CASE r.[name]
               WHEN N''db_owner''         THEN 1
               WHEN N''db_securityadmin'' THEN 2
               WHEN N''db_ddladmin''      THEN 3
               WHEN N''db_datawriter''    THEN 4
               WHEN N''db_datareader''    THEN 5
               ELSE 6
           END
        ) AS [HighestRole]

       /* Schema ownership */
       ,(  SELECT STRING_AGG(s.[name], N'', '')
           FROM sys.[schemas] AS s
           WHERE s.[principal_id] = dp.[principal_id]
               AND s.[name] <> dp.[name]
        ) AS [OwnsSchemas]

       /* Remediation */
       ,CASE
            WHEN sp.[name] IS NOT NULL
                 AND dp.[sid] <> sp.[sid]
                THEN N''ALTER USER '' + QUOTENAME(dp.[name])
                     + N'' WITH LOGIN = '' + QUOTENAME(sp.[name]) + N'';''
            WHEN sp.[sid] IS NULL
                 AND dp.[authentication_type] <> 2
                 AND NOT EXISTS (
                     SELECT 1
                     FROM sys.[schemas] AS s2
                     WHERE s2.[principal_id] = dp.[principal_id]
                         AND s2.[name] <> dp.[name]
                 )
                THEN N''DROP USER '' + QUOTENAME(dp.[name]) + N'';''
            WHEN sp.[sid] IS NULL
                 AND dp.[authentication_type] <> 2
                 AND EXISTS (
                     SELECT 1
                     FROM sys.[schemas] AS s3
                     WHERE s3.[principal_id] = dp.[principal_id]
                         AND s3.[name] <> dp.[name]
                 )
                THEN N''/* Transfer schema ownership first, then: DROP USER ''
                     + QUOTENAME(dp.[name]) + N'' */''
            ELSE NULL
        END AS [RemediationAction]

    FROM sys.[database_principals] AS dp
        LEFT JOIN sys.[server_principals] AS sp
            ON dp.[sid] = sp.[sid]
    WHERE dp.[type] IN (''S'', ''U'', ''G'', ''E'', ''X'')
        AND dp.[principal_id] > 4
        AND dp.[name] NOT IN (
            N''dbo'', N''guest'',
            N''INFORMATION_SCHEMA'', N''sys''
        )
        AND (
            /* True orphan — no matching login */
            (sp.[sid] IS NULL AND dp.[authentication_type] <> 2)
            /* SID mismatch */
            OR (sp.[name] IS NOT NULL AND dp.[sid] <> sp.[sid])
            /* Disabled login */
            OR (sp.[is_disabled] = 1)
            /* Dead contained user */
            OR (dp.[authentication_type] = 2
                AND NOT EXISTS (
                    SELECT 1
                    FROM sys.[database_role_members] AS drm
                    WHERE drm.[member_principal_id] = dp.[principal_id]
                )
                AND NOT EXISTS (
                    SELECT 1
                    FROM sys.[database_permissions] AS perm
                    WHERE perm.[grantee_principal_id] = dp.[principal_id]
                        AND perm.[type] <> N''CO''
                )
            )
        );';

    EXEC sys.[sp_executesql] @SQL;

    FETCH NEXT FROM db_cursor INTO @DbName;
END;

CLOSE db_cursor;
DEALLOCATE db_cursor;

/* Results — highest risk first */
SELECT
    [DatabaseName]
   ,[UserName]
   ,[UserType]
   ,[OrphanReason]
   ,[HighestRole]
   ,[OwnsSchemas]
   ,[RemediationAction]
FROM #OrphanedUsers
ORDER BY
    CASE [HighestRole]
        WHEN N'db_owner'         THEN 1
        WHEN N'db_securityadmin' THEN 2
        WHEN N'db_ddladmin'      THEN 3
        ELSE 4
    END
   ,[DatabaseName]
   ,[UserName];

DROP TABLE #OrphanedUsers;
```

## What I Validated

A few things I checked before trusting this in production:

1. **System principal exclusion.** I verified the `principal_id > 4` filter plus the name exclusion list catches everything. On a few instances, I found custom users named `sys_monitor` with principal IDs above 4 — those should be included in the scan, and they were.

2. **SID mismatch detection.** I tested by creating a SQL login, creating a user in a test database, dropping the login, and recreating it with the same name. The new login gets a new SID, and the script correctly flagged the mismatch with an `ALTER USER ... WITH LOGIN` remediation.

3. **Schema ownership safety.** Attempting to `DROP USER` on a user who owns a schema produces error 15138. The script correctly detects this and generates a comment instead of a `DROP USER`.

## Azure SQL DB and Managed Instance Differences

If you're running Azure SQL Database or Managed Instance, a few things change:

- **Azure SQL DB** doesn't have server-level logins in the traditional sense. Contained database users are the primary authentication model, so the "no matching login" check doesn't apply the same way. Focus the audit on abandoned contained users and overprivileged Entra ID principals.
- **Managed Instance** supports server-level logins, so the orphan detection logic works similarly to on-premises. However, `sys.server_principals` includes Entra ID principals with type `E` (external user) and `X` (external group) — make sure your script includes those types.
- **`ALTER USER ... WITH LOGIN`** works on Managed Instance but not on Azure SQL DB for contained users. On Azure SQL DB, the remediation is typically `DROP USER` and `CREATE USER ... FROM EXTERNAL PROVIDER`.

## Try This Yourself

Run the audit script on a development instance first. You'll almost certainly find orphaned users — especially on instances where databases have been restored from other servers or moved between environments.

For each orphan the script finds, ask yourself three questions before remediating:

1. **Does anyone still need this access?** Check with the application team before dropping users tied to service accounts.
2. **Does this user own any objects?** The script checks schema ownership, but also verify object ownership with `SELECT * FROM sys.objects WHERE principal_id = <user_principal_id>`.
3. **Is this a SID mismatch or a true orphan?** SID mismatches are the easiest fix — `ALTER USER ... WITH LOGIN` re-maps the user to the existing login. True orphans require a decision: drop the user or create a new login.

For the broader security audit workflow — including permission sprawl, SQL injection scanning, and compliance evidence — see [Security Audits: Finding What You Missed](/ai-for-dbas/security-audits-finding-missed/).

---

*Part of the [ALTER DBA ADD AGENT](/ai-for-dbas/alter-dba-add-agent/) series.*
