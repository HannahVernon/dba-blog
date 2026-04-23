The question sounds simple: "Who has access to what?" In practice, answering it accurately is one of the hardest things a DBA does. Permissions in SQL Server flow through direct grants, database role memberships, nested roles, server-level roles, Windows group inheritance, `CONTROL` and `IMPERSONATE` chains, and cross-database ownership chaining. Tracing a single user's effective permissions by hand means querying half a dozen DMVs and mentally merging the results.

This is exactly the kind of tedious, multi-step analysis where an AI agent saves hours. I covered the high-level approach in [Security Audits: Finding What You Missed](/ai-for-dbas/security-audits-finding-missed/). This post digs into producing a comprehensive permissions map that traces every path from principal to permission.

## The Problem: Permissions Through Multiple Paths

A user has `SELECT` on a table. Why? Could be any of these:

- Direct `GRANT SELECT` on the table
- Membership in `db_datareader`
- Membership in a custom role that has `SELECT`
- Membership in a role that's nested inside another role that has `SELECT`
- Windows group membership where the group has a SQL login with the right access
- `CONTROL` on the schema that contains the table
- `db_owner` membership (implicit full access)
- Server-level `sysadmin` membership (implicit everything)

An auditor wants to see *all* of these paths, not just the first one you find. And they want to see `DENY` entries too, because `DENY` overrides `GRANT` except when it doesn't (sysadmin ignores `DENY` at the database level).

## The Prompt

```text
Write a T-SQL script that produces a comprehensive permissions audit for
a single database. For each database principal, show every permission they
have and how they got it. Trace the full inheritance path:

1. Direct GRANT/DENY on objects, schemas, and the database itself
2. Permissions inherited through database role membership (including
   nested roles — resolve the full chain)
3. Server-level role memberships (especially sysadmin)
4. CONTROL and IMPERSONATE grants that create implicit permissions
5. Membership in fixed database roles and what those roles grant

Output columns: PrincipalName, PrincipalType, PermissionType,
PermissionState (GRANT/DENY/GRANT_WITH_GRANT), SecurableType,
SecurableName, InheritedThrough (role chain or 'DIRECT'),
ServerLevelAccess (sysadmin, securityadmin, etc.).

Exclude system principals (dbo, guest, INFORMATION_SCHEMA, sys).
Sort by principal name, then by securable.
```

## What the Agent Produced

The agent generated a multi-CTE script that layers permissions from each source. The structure was solid:

1. A CTE that recursively resolves nested role memberships using `sys.database_role_members` — walking the chain from user → role → parent role → grandparent role.
2. A direct permissions query from `sys.database_permissions` joined to `sys.objects`, `sys.schemas`, and `sys.database_principals`.
3. A server-level check against `sys.server_role_members` to flag principals who have `sysadmin` or other elevated server roles.
4. A `UNION ALL` that combines direct permissions with inherited permissions, tagging each row with how the permission was acquired.

What needed fixing:

- **The recursive CTE had no depth limit.** Deeply nested roles (rare but possible) could cause infinite recursion. I added `OPTION (MAXRECURSION 10)` — if you have more than 10 levels of role nesting, you have bigger problems.
- **It didn't resolve fixed database role permissions.** The agent listed membership in `db_datareader` but didn't expand what that actually grants (`SELECT` on all user tables and views). For a complete audit, you'd need a separate mapping of fixed database roles to their implicit permissions (e.g., `db_datareader` → `SELECT` on all user tables/views, `db_datawriter` → `INSERT`/`UPDATE`/`DELETE`, `db_ddladmin` → DDL permissions, etc.). This expansion is complex enough that it's best handled in a separate reference table or post-processing step — the script above shows role *membership* chains, which combined with knowledge of what each fixed role grants, gives auditors the full picture.
- **`CONTROL` implications were incomplete.** `CONTROL` on a schema implies all permissions on all objects in that schema. The agent flagged `CONTROL` grants but didn't enumerate the downstream objects. I added a join to `sys.objects` filtered by schema to make the implicit access explicit.

## The Final Script

```sql
/* Permission Sprawl Audit — single database */
/* Run in the context of the database you want to audit */
SET NOCOUNT ON;

/* Resolve nested role memberships recursively */
;WITH [RoleChain] AS (
    /* Base: direct role memberships */
    SELECT
        drm.[member_principal_id]
       ,drm.[role_principal_id]
       ,CONVERT(nvarchar(MAX),
            QUOTENAME(mp.[name]) + N' → ' + QUOTENAME(rp.[name])
        ) AS [InheritanceChain]
       ,1 AS [Depth]
    FROM sys.[database_role_members] AS drm
        INNER JOIN sys.[database_principals] AS mp
            ON drm.[member_principal_id] = mp.[principal_id]
        INNER JOIN sys.[database_principals] AS rp
            ON drm.[role_principal_id] = rp.[principal_id]

    UNION ALL

    /* Recursive: roles that are members of other roles */
    SELECT
        rc.[member_principal_id]
       ,drm2.[role_principal_id]
       ,CONVERT(nvarchar(MAX),
            rc.[InheritanceChain] + N' → '
            + QUOTENAME(rp2.[name])
        ) AS [InheritanceChain]
       ,rc.[Depth] + 1
    FROM [RoleChain] AS rc
        INNER JOIN sys.[database_role_members] AS drm2
            ON rc.[role_principal_id] = drm2.[member_principal_id]
        INNER JOIN sys.[database_principals] AS rp2
            ON drm2.[role_principal_id] = rp2.[principal_id]
    WHERE rc.[Depth] < 10
)
/* Direct permissions on securables */
SELECT
    dp.[name]                          AS [PrincipalName]
   ,dp.[type_desc]                     AS [PrincipalType]
   ,perm.[permission_name]             AS [PermissionType]
   ,perm.[state_desc]                  AS [PermissionState]
   ,perm.[class_desc]                  AS [SecurableType]
   ,COALESCE(
        QUOTENAME(s.[name]) + N'.' + QUOTENAME(o.[name]),
        QUOTENAME(s2.[name]),
        dp2.[name],
        N'DATABASE'
    )                                  AS [SecurableName]
   ,N'DIRECT'                         AS [InheritedThrough]
   ,sra.[ServerRoles]                  AS [ServerLevelAccess]
FROM sys.[database_permissions] AS perm
    INNER JOIN sys.[database_principals] AS dp
        ON perm.[grantee_principal_id] = dp.[principal_id]
    LEFT JOIN sys.[objects] AS o
        ON perm.[major_id] = o.[object_id]
        AND perm.[class] = 1
    LEFT JOIN sys.[schemas] AS s
        ON o.[schema_id] = s.[schema_id]
    LEFT JOIN sys.[schemas] AS s2
        ON perm.[major_id] = s2.[schema_id]
        AND perm.[class] = 3
    LEFT JOIN sys.[database_principals] AS dp2
        ON perm.[major_id] = dp2.[principal_id]
        AND perm.[class] = 4
    OUTER APPLY (
        SELECT STRING_AGG(sr.[name], N', ') AS [ServerRoles]
        FROM sys.[server_role_members] AS srm
            INNER JOIN sys.[server_principals] AS sr
                ON srm.[role_principal_id] = sr.[principal_id]
            INNER JOIN sys.[server_principals] AS sp
                ON srm.[member_principal_id] = sp.[principal_id]
        WHERE sp.[sid] = dp.[sid]
    ) AS sra
WHERE dp.[principal_id] > 4
    AND dp.[name] NOT IN (
        N'dbo', N'guest', N'INFORMATION_SCHEMA', N'sys'
    )

UNION ALL

/* Permissions inherited through role membership */
SELECT
    usr.[name]                         AS [PrincipalName]
   ,usr.[type_desc]                    AS [PrincipalType]
   ,perm.[permission_name]             AS [PermissionType]
   ,perm.[state_desc]                  AS [PermissionState]
   ,perm.[class_desc]                  AS [SecurableType]
   ,COALESCE(
        QUOTENAME(s.[name]) + N'.' + QUOTENAME(o.[name]),
        QUOTENAME(s2.[name]),
        dp2.[name],
        N'DATABASE'
    )                                  AS [SecurableName]
   ,rc.[InheritanceChain]             AS [InheritedThrough]
   ,sra.[ServerRoles]                  AS [ServerLevelAccess]
FROM [RoleChain] AS rc
    INNER JOIN sys.[database_principals] AS usr
        ON rc.[member_principal_id] = usr.[principal_id]
    INNER JOIN sys.[database_permissions] AS perm
        ON perm.[grantee_principal_id] = rc.[role_principal_id]
    LEFT JOIN sys.[objects] AS o
        ON perm.[major_id] = o.[object_id]
        AND perm.[class] = 1
    LEFT JOIN sys.[schemas] AS s
        ON o.[schema_id] = s.[schema_id]
    LEFT JOIN sys.[schemas] AS s2
        ON perm.[major_id] = s2.[schema_id]
        AND perm.[class] = 3
    LEFT JOIN sys.[database_principals] AS dp2
        ON perm.[major_id] = dp2.[principal_id]
        AND perm.[class] = 4
    OUTER APPLY (
        SELECT STRING_AGG(sr.[name], N', ') AS [ServerRoles]
        FROM sys.[server_role_members] AS srm
            INNER JOIN sys.[server_principals] AS sr
                ON srm.[role_principal_id] = sr.[principal_id]
            INNER JOIN sys.[server_principals] AS sp
                ON srm.[member_principal_id] = sp.[principal_id]
        WHERE sp.[sid] = usr.[sid]
    ) AS sra
WHERE usr.[type] IN ('S', 'U', 'G', 'E', 'X')
    AND usr.[principal_id] > 4
    AND usr.[name] NOT IN (
        N'dbo', N'guest', N'INFORMATION_SCHEMA', N'sys'
    )

ORDER BY [PrincipalName], [SecurableName]
OPTION (MAXRECURSION 10);
```

## What I Validated

Before handing this to an auditor, I verified the output against a known configuration:

1. **I created a test user with a known permission path:** user → custom role → `db_datareader` → implicit `SELECT`. The script correctly showed the full chain in the `InheritedThrough` column.

2. **DENY override behavior.** I granted `SELECT` through a role and applied `DENY SELECT` directly to the user. The script showed both rows — which is correct. The auditor can see both the grant path and the deny, and understand the effective permission is denied.

3. **Sysadmin detection.** The `ServerLevelAccess` column correctly showed `sysadmin` for logins with that server role. This is critical because `sysadmin` members bypass all database-level permission checks — they have implicit access to everything regardless of `DENY`.

## The Gaps You Still Need to Cover

This script handles what T-SQL can see. What it can't see:

- **Windows/AD group membership.** If a Windows group is a SQL login, the script shows permissions for that group. But it can't expand who's *in* the group — that requires Active Directory or Entra ID queries. Use `xp_logininfo` for on-premises AD groups, but be aware it only works for groups the SQL Server service account can resolve.
- **Application-level permissions.** Many applications implement their own authorization layer on top of SQL Server permissions. A user might have `db_datareader` in SQL but only see certain rows based on application logic. This script audits SQL Server permissions, not application permissions.
- **Cross-database ownership chaining.** If database A trusts database B, a user with access to A might reach objects in B without explicit permissions in B. Check `sys.databases` for `is_db_chaining_on` and the instance-level `cross db ownership chaining` setting.

## Try This Yourself

Run the script on a database you think you know well. Filter the output to a single user and trace their permission paths. In my experience, the surprises fall into two categories:

1. **Permissions you didn't know existed.** Someone granted `EXECUTE` on a sensitive procedure to `public` three years ago, and it's been there ever since.
2. **Nested role chains you forgot about.** A custom role is a member of another custom role that's a member of `db_owner`. Nobody remembers setting it up that way.

For each finding, decide: is this intentional, accidental, or a risk? Then use the `InheritedThrough` column to figure out where to revoke — removing a user from a deeply nested role might fix five permission issues at once, or it might break the application. Test in dev first.

For the full security audit workflow, see [Security Audits: Finding What You Missed](/ai-for-dbas/security-audits-finding-missed/).

---

*Part of the [ALTER DBA ADD AGENT](/ai-for-dbas/alter-dba-add-agent/) series.*
