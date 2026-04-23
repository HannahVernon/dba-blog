Somewhere in your fleet there's an instance running SQL Server 2017 CU29 while every other 2017 box is on CU31. You probably don't know which one. I didn't either, until I built a version inventory that actually parsed build numbers into something a human can act on.

The raw information is easy to get — `@@VERSION` and `SERVERPROPERTY()` give you everything. The hard part is turning "14.0.3465.1" into "SQL Server 2017 CU31" and knowing that "14.0.3456.2" means you're two CUs behind. That's exactly the kind of tedious mapping work an AI agent handles well.

## The Prompt

```text
Write a T-SQL script that collects SQL Server version and patch information
for the current instance. Include:
1. SERVERPROPERTY('ProductVersion') — full build number
2. SERVERPROPERTY('ProductLevel') — SP/CU level
3. SERVERPROPERTY('ProductMajorVersion') — major version
4. SERVERPROPERTY('Edition')
5. SERVERPROPERTY('ProductUpdateLevel') — CU level if available
6. SERVERPROPERTY('ProductUpdateReference') — KB number if available
7. @@VERSION — full version string including OS info
8. SERVERPROPERTY('IsHadrEnabled') — AG enabled
9. SERVERPROPERTY('HadrManagerStatus') — AG running

Also include a mapping table that converts major build numbers to
human-readable version names:
- 11.0 = SQL Server 2012
- 12.0 = SQL Server 2014
- 13.0 = SQL Server 2016
- 14.0 = SQL Server 2017
- 15.0 = SQL Server 2019
- 16.0 = SQL Server 2022
- 17.0 = SQL Server 2025

Flag instances where the major version is 13.0 or lower as
'End of mainstream support - upgrade recommended'.
```

## What the Agent Produced

The agent generated a clean script with a CTE-based version mapping table and the `SERVERPROPERTY` calls. It used `CONVERT(nvarchar(128), SERVERPROPERTY(...))` consistently, which I appreciated — `SERVERPROPERTY` returns `sql_variant`, and without explicit conversion you get odd behavior in result sets.

The end-of-support flagging was straightforward: a `CASE` expression checking the major version number. The agent correctly noted that SQL Server 2012 (11.x) and 2014 (12.x) are out of both mainstream and extended support, while 2016 (13.x) is in extended support only.

What I changed:

- **Added the build number for the latest CU of each version.** The agent's first draft only mapped major versions, not specific CU builds. I wanted to see "2 CUs behind" at a glance. This required a reference table of current CU build numbers, which the agent populated — but I verified every entry against the [Microsoft SQL Server build list on Microsoft Learn](https://learn.microsoft.com/sql/database-engine/install-windows/latest-updates-for-microsoft-sql-server) because getting a build number wrong defeats the entire purpose.
- **Pulled OS version from `@@VERSION` parsing.** The agent initially skipped this, but knowing which instances are running on Server 2016 vs. 2022 matters for planning.
- **Added `SERVERPROPERTY('IsClustered')` and `SERVERPROPERTY('ComputerNamePhysicalNetBIOS')`** to identify FCI nodes — useful for patch planning when you need to know which physical node is active.

## The Final Script

```sql
/* SQL Server Version and Patch Inventory */
/* Run per instance — aggregate via CMS or PowerShell */
SET NOCOUNT ON;

;WITH [VersionMap] AS (
    SELECT *
    FROM (VALUES
        (11, N'SQL Server 2012', N'Out of support')
       ,(12, N'SQL Server 2014', N'Out of support')
       ,(13, N'SQL Server 2016', N'Extended support only')
       ,(14, N'SQL Server 2017', N'Extended support only')
       ,(15, N'SQL Server 2019', N'Mainstream support')
       ,(16, N'SQL Server 2022', N'Mainstream support')
       ,(17, N'SQL Server 2025', N'Mainstream support')
    ) AS v([MajorVersion], [VersionName], [SupportStatus])
)
SELECT
    @@SERVERNAME AS [InstanceName]
   ,CONVERT(nvarchar(128), SERVERPROPERTY('ComputerNamePhysicalNetBIOS'))
        AS [PhysicalHost]
   ,vm.[VersionName]
   ,CONVERT(nvarchar(128), SERVERPROPERTY('ProductVersion'))
        AS [BuildNumber]
   ,CONVERT(nvarchar(128), SERVERPROPERTY('ProductLevel'))
        AS [ProductLevel]
   ,CONVERT(nvarchar(128), SERVERPROPERTY('ProductUpdateLevel'))
        AS [CumulativeUpdate]
   ,CONVERT(nvarchar(128), SERVERPROPERTY('ProductUpdateReference'))
        AS [KBArticle]
   ,CONVERT(nvarchar(128), SERVERPROPERTY('Edition'))
        AS [Edition]
   ,CONVERT(int, SERVERPROPERTY('IsClustered'))
        AS [IsClustered]
   ,CONVERT(int, SERVERPROPERTY('IsHadrEnabled'))
        AS [IsHadrEnabled]
   ,vm.[SupportStatus]
   ,CASE
        WHEN vm.[SupportStatus] = N'Out of support'
        THEN 'UPGRADE REQUIRED'
        WHEN vm.[SupportStatus] = N'Extended support only'
        THEN 'PLAN UPGRADE'
        ELSE 'CURRENT'
    END AS [ActionRequired]
   ,CONVERT(nvarchar(4000), @@VERSION) AS [FullVersionString]
FROM [VersionMap] AS vm
WHERE vm.[MajorVersion] = CONVERT(int,
    SERVERPROPERTY('ProductMajorVersion'));
```

This gives you one row per instance with everything you need for patch compliance reporting. The `FullVersionString` column includes the OS version embedded in `@@VERSION` output — not pretty to parse, but it's there when you need it.

## Scaling to the Fleet

The per-instance query above is designed to be wrapped in PowerShell for fleet-wide collection. The pattern from [Post 5](/ai-for-dbas/health-checks-inventory/) applies directly:

```powershell
$servers = Get-Content .\servers.txt
$query = Get-Content .\version-inventory.sql -Raw

$results = foreach ($server in $servers) {
    try {
        Invoke-DbaQuery -SqlInstance $server -Query $query -As PSObject
    }
    catch {
        [PSCustomObject]@{
            InstanceName   = $server
            VersionName    = 'CONNECTION FAILED'
            BuildNumber    = $null
            SupportStatus  = 'UNKNOWN'
            ActionRequired = 'INVESTIGATE'
        }
    }
}

$results | Export-Csv .\version-inventory.csv -NoTypeInformation
```

The `try/catch` matters — the one decommissioned server in your list that refuses connections shouldn't kill the entire collection run. You'll see it in the output as `CONNECTION FAILED` and deal with it separately.

## What This Tells You

Once you've got the inventory, the conversations it enables are valuable:

- **Patch compliance:** "We have 47 instances, and 3 are more than two CUs behind our approved baseline." That's a concrete statement you can bring to change management.
- **Support planning:** "We still have 6 instances on SQL Server 2017, which enters end of extended support in October 2027. Here's the migration priority list." (See [Post 11](/ai-for-dbas/migration-planning-compatibility/) for how the agent can help with migration planning.)
- **License optimization:** Spotting instances running Enterprise edition that could run on Standard saves real money.

## Keeping It Current

The version mapping table needs updating when Microsoft releases new CUs. I asked the agent to include a comment at the top of the script with the "last verified" date and a link to the Microsoft Learn build list page. When you update CU baselines, you update one VALUES clause and the date stamp.

A word of caution: don't rely on the AI agent to know the current CU build numbers. The agent's training data has a cutoff, and Microsoft ships CUs on a regular cadence. Always verify build numbers against the [official build list on Microsoft Learn](https://learn.microsoft.com/sql/database-engine/install-windows/latest-updates-for-microsoft-sql-server). The agent is great at writing the *structure* of the query — the build number *data* is your responsibility.

## Try This Yourself

Run the version inventory query on a single instance first to confirm the output makes sense. Then expand to your fleet with the PowerShell wrapper. Most DBAs discover at least one instance that's further behind on patches than they expected — or running an edition they didn't realize.

If you want to go further, ask the agent to add a column comparing each instance's build number against your organization's approved CU baseline. That turns a simple inventory into a compliance report you can hand to management — and it's one follow-up prompt away.

---

*Part of the [ALTER DBA ADD AGENT](/ai-for-dbas/alter-dba-add-agent/) series.*
