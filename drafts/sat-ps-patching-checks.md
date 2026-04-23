SQL Server patching should be boring. Apply the CU, restart the service, verify everything's back to normal, move on. In practice, it's the "verify everything's back to normal" part that takes the longest — and it's where things go wrong when you rush. A service that didn't restart. An AG replica that's stuck in RESOLVING. A linked server that lost its credentials. A Resource Governor pool that reverted to defaults.

I've been burned enough times that I now capture a comprehensive baseline before patching and compare it afterwards. Doing this manually takes 45 minutes per instance. I asked the agent to automate it.

```text
Write a matched pair of PowerShell scripts for SQL Server patching
validation using dbatools:

Script 1 — Pre-Patch Baseline (Save-PatchBaseline):
Capture the following from a SQL Server instance and export to a JSON file:
- SQL Server build number and edition
- All Windows services related to SQL Server (name, status, start type)
- All database names, states, recovery models, and compatibility levels
- AG health: AG name, replica names, roles, sync states, sync health
- All SQL Agent jobs: name, enabled status, last run outcome, schedule
- All linked servers and their provider strings
- All server-level logins (name, type, is_disabled)
- All endpoints (name, type, state)
- Resource Governor configuration: pools, groups, and classifier function
- Server configuration (sp_configure values that differ from defaults)

Script 2 — Post-Patch Validation (Test-PatchBaseline):
Read the baseline JSON and compare against the current state. Report:
- Build number changed (expected — confirm it matches the target CU)
- Any service not running that was running before
- Any database offline, restoring, or in a different state
- Any AG replica not synchronized or not connected
- Any SQL Agent job that changed from enabled to disabled
- Any linked server missing or with a different provider
- Any login missing or with a changed disabled state
- Any endpoint in a different state
- Any Resource Governor setting that changed
- Any sp_configure value that changed unexpectedly

Output a pass/fail summary with details for each check. Return a
non-zero exit code if any critical check fails (services down,
databases offline, AG unhealthy).

Use splatting, Write-Verbose for logging, and try/catch throughout.
```

## What the Agent Produced

The agent generated two clean functions: `Save-PatchBaseline` and `Test-PatchBaseline`. The baseline export was thorough — it used a combination of dbatools cmdlets and direct DMV queries to capture everything.

What it got right immediately:

- **JSON export/import.** The baseline saves as a single JSON file with a timestamp and server name in the filename. Clean and portable.
- **The comparison logic** was property-by-property, not just "did the count change." It detected a specific login going from enabled to disabled, not just "login count went from 42 to 41."
- **Exit codes.** Critical failures (services down, databases offline) returned exit code 1. Warnings (config changes) returned exit code 0 with console warnings. This integrates nicely with SCCM or other patching orchestration.

What needed fixing:

- **It missed certificate-based endpoints.** The initial version captured endpoint name and state but not the authentication method. If patching somehow disrupts a certificate binding, I want to know.
- **AG comparison was too coarse.** It compared replica roles — but after patching and restarting, the replica might have failed over. A role change isn't a failure; it's expected. I needed the comparison to distinguish between "AG is healthy with roles swapped" and "AG is broken."
- **No smoke test.** Verifying metadata is good, but I also want to run a quick connectivity test — can I connect, execute a query, and get results?

## The Iteration

```text
Add these improvements:
1. For AG checks, don't flag a role swap as a failure. Instead, report
   it as informational ("roles changed — verify this was intentional")
   and only flag FAIL if a replica is DISCONNECTED, NOT SYNCHRONIZING,
   or a database is NOT JOINED.
2. Add a -SmokeTestQuery parameter that accepts a hashtable of
   database/query pairs. After all other checks, run each query and
   verify it returns results without error.
3. For the build number check, add a -ExpectedBuild parameter. If
   provided, verify the post-patch build matches. If not provided,
   just report the change.
4. Capture and compare database-level settings: is_auto_close_on,
   is_auto_shrink_on, page_verify_option_desc, and target_recovery_time.
```

## The Final Solution

```powershell
function Save-PatchBaseline {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)]
        [string]$SqlInstance,

        [Parameter(Mandatory)]
        [string]$OutputPath
    )

    Write-Verbose "Capturing baseline for $SqlInstance"
    $baseline = [ordered]@{}
    $baseline['CaptureTime'] = (Get-Date).ToString('yyyy-MM-dd HH:mm:ss')
    $baseline['SqlInstance'] = $SqlInstance

    try {
        /* Build number and edition */
        $serverInfo = Get-DbaInstanceProperty -SqlInstance $SqlInstance -EnableException
        $build = Get-DbaBuild -SqlInstance $SqlInstance -EnableException
        $baseline['Build'] = @{
            ProductVersion = $build.Build
            CU             = $build.CULevel
            SPLevel        = $build.SPLevel
            Edition        = ($serverInfo | Where-Object { $_.Name -eq 'Edition' }).Value
        }
        Write-Verbose "  Build: $($build.Build) ($($build.CULevel))"

        /* Windows services */
        $services = Get-DbaService -ComputerName $SqlInstance -EnableException |
            Select-Object ServiceName, State, StartMode, ServiceType
        $baseline['Services'] = @($services)
        Write-Verbose "  Services: $($services.Count)"

        /* Databases */
        $dbQuery = @"
            /* Database state baseline */
            SELECT
                d.[name]
               ,d.[state_desc]
               ,d.[recovery_model_desc]
               ,d.[compatibility_level]
               ,d.[is_auto_close_on]
               ,d.[is_auto_shrink_on]
               ,d.[page_verify_option_desc]
               ,d.[target_recovery_time_in_seconds]
            FROM sys.[databases] AS d
            ORDER BY d.[name];
"@
        $databases = Invoke-DbaQuery -SqlInstance $SqlInstance `
            -Query $dbQuery -EnableException
        $baseline['Databases'] = @($databases)
        Write-Verbose "  Databases: $($databases.Count)"

        /* Availability Groups */
        $agQuery = @"
            /* AG replica state baseline */
            SELECT
                ag.[name] AS [AgName]
               ,r.[replica_server_name]
               ,rs.[role_desc]
               ,rs.[connected_state_desc]
               ,rs.[synchronization_health_desc]
               ,drs.[database_name]
               ,drs.[synchronization_state_desc]
               ,drs.[is_joined]
            FROM sys.[availability_groups] AS ag
                INNER JOIN sys.[availability_replicas] AS r
                    ON ag.[group_id] = r.[group_id]
                INNER JOIN sys.dm_hadr_availability_replica_states AS rs
                    ON r.[replica_id] = rs.[replica_id]
                LEFT JOIN sys.dm_hadr_database_replica_states AS drs
                    ON drs.[replica_id] = r.[replica_id]
            ORDER BY ag.[name], r.[replica_server_name];
"@
        try {
            $agState = Invoke-DbaQuery -SqlInstance $SqlInstance `
                -Query $agQuery -EnableException
            $baseline['AvailabilityGroups'] = @($agState)
            Write-Verbose "  AG replicas: $($agState.Count)"
        }
        catch {
            $baseline['AvailabilityGroups'] = @()
            Write-Verbose "  No Availability Groups configured"
        }

        /* SQL Agent jobs */
        $jobs = Get-DbaAgentJob -SqlInstance $SqlInstance -EnableException |
            Select-Object Name, IsEnabled, LastRunOutcome, LastRunDate, HasSchedule
        $baseline['AgentJobs'] = @($jobs)
        Write-Verbose "  Agent jobs: $($jobs.Count)"

        /* Linked servers */
        $linkedServers = Get-DbaLinkedServer -SqlInstance $SqlInstance -EnableException |
            Select-Object Name, DataSource, ProviderName, ProductName
        $baseline['LinkedServers'] = @($linkedServers)
        Write-Verbose "  Linked servers: $($linkedServers.Count)"

        /* Logins */
        $loginQuery = @"
            /* Server login baseline */
            SELECT
                [name]
               ,[type_desc]
               ,[is_disabled]
               ,[default_database_name]
            FROM sys.[server_principals]
            WHERE [type] IN ('S', 'U', 'G')
                AND [name] NOT LIKE N'##%'
            ORDER BY [name];
"@
        $logins = Invoke-DbaQuery -SqlInstance $SqlInstance `
            -Query $loginQuery -EnableException
        $baseline['Logins'] = @($logins)
        Write-Verbose "  Logins: $($logins.Count)"

        /* Endpoints */
        $endpointQuery = @"
            /* Endpoint baseline */
            SELECT [name], [type_desc], [state_desc], [protocol_desc]
            FROM sys.[endpoints]
            WHERE [type] <> 2; /* exclude TSQL default */
"@
        $endpoints = Invoke-DbaQuery -SqlInstance $SqlInstance `
            -Query $endpointQuery -EnableException
        $baseline['Endpoints'] = @($endpoints)
        Write-Verbose "  Endpoints: $($endpoints.Count)"

        /* Resource Governor */
        $rgQuery = @"
            /* Resource Governor configuration */
            SELECT
                rp.[name] AS [PoolName]
               ,rp.[min_cpu_percent]
               ,rp.[max_cpu_percent]
               ,rp.[cap_cpu_percent]
               ,wg.[name] AS [WorkloadGroup]
               ,wg.[max_dop]
               ,wg.[request_max_memory_grant_percent]
            FROM sys.[resource_governor_resource_pools] AS rp
                LEFT JOIN sys.[resource_governor_workload_groups] AS wg
                    ON rp.[pool_id] = wg.[pool_id]
            ORDER BY rp.[name], wg.[name];
"@
        $rgConfig = Invoke-DbaQuery -SqlInstance $SqlInstance `
            -Query $rgQuery -EnableException
        $baseline['ResourceGovernor'] = @($rgConfig)

        /* sp_configure */
        $spConfig = Get-DbaSpConfigure -SqlInstance $SqlInstance -EnableException |
            Where-Object { $_.RunningValue -ne $_.DefaultValue } |
            Select-Object DisplayName, ConfiguredValue, RunningValue
        $baseline['SpConfigure'] = @($spConfig)
        Write-Verbose "  Non-default sp_configure: $($spConfig.Count)"
    }
    catch {
        Write-Error "Failed to capture baseline: $($_.Exception.Message)"
        return
    }

    $filename = "baseline_$($SqlInstance -replace '\\', '_')_$(Get-Date -Format 'yyyyMMdd_HHmmss').json"
    $fullPath = Join-Path -Path $OutputPath -ChildPath $filename

    $baseline | ConvertTo-Json -Depth 10 | Set-Content -Path $fullPath -Encoding UTF8
    Write-Verbose "Baseline saved to $fullPath"
    Write-Output $fullPath
}

function Test-PatchBaseline {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)]
        [string]$SqlInstance,

        [Parameter(Mandatory)]
        [string]$BaselinePath,

        [string]$ExpectedBuild,

        [hashtable]$SmokeTestQueries
    )

    $baseline = Get-Content -Path $BaselinePath -Raw | ConvertFrom-Json
    $checkResults = [System.Collections.Generic.List[PSCustomObject]]::new()
    $criticalFailure = $false

    function Add-CheckResult {
        param([string]$Category, [string]$Status, [string]$Detail)
        $checkResults.Add([PSCustomObject]@{
            Category = $Category
            Status   = $Status
            Detail   = $Detail
        })
        if ($Status -eq 'FAIL') { $Script:criticalFailure = $true }
    }

    Write-Verbose "Validating $SqlInstance against baseline from $($baseline.CaptureTime)"

    /* ── Build Number ── */
    $currentBuild = Get-DbaBuild -SqlInstance $SqlInstance -EnableException
    if ($ExpectedBuild -and $currentBuild.Build -ne $ExpectedBuild) {
        Add-CheckResult -Category 'Build' -Status 'FAIL' `
            -Detail "Expected $ExpectedBuild, found $($currentBuild.Build)"
    }
    elseif ($currentBuild.Build -ne $baseline.Build.ProductVersion) {
        Add-CheckResult -Category 'Build' -Status 'INFO' `
            -Detail "Changed from $($baseline.Build.ProductVersion) to $($currentBuild.Build) ($($currentBuild.CULevel))"
    }
    else {
        Add-CheckResult -Category 'Build' -Status 'WARN' `
            -Detail "Build unchanged at $($currentBuild.Build) — patch may not have applied"
    }

    /* ── Services ── */
    $currentServices = Get-DbaService -ComputerName $SqlInstance -EnableException
    foreach ($svc in $baseline.Services) {
        $current = $currentServices | Where-Object { $_.ServiceName -eq $svc.ServiceName }
        if (-not $current) {
            Add-CheckResult -Category 'Services' -Status 'FAIL' `
                -Detail "Service missing: $($svc.ServiceName)"
        }
        elseif ($svc.State -eq 'Running' -and $current.State -ne 'Running') {
            Add-CheckResult -Category 'Services' -Status 'FAIL' `
                -Detail "$($svc.ServiceName) was Running, now $($current.State)"
        }
        else {
            Add-CheckResult -Category 'Services' -Status 'PASS' `
                -Detail "$($svc.ServiceName) is $($current.State)"
        }
    }

    /* ── Databases ── */
    $currentDbs = Invoke-DbaQuery -SqlInstance $SqlInstance -Query @"
        SELECT [name], [state_desc], [recovery_model_desc],
               [compatibility_level], [is_auto_close_on],
               [is_auto_shrink_on], [page_verify_option_desc],
               [target_recovery_time_in_seconds]
        FROM sys.[databases]
        ORDER BY [name];
"@ -EnableException

    foreach ($db in $baseline.Databases) {
        $current = $currentDbs | Where-Object { $_.name -eq $db.name }
        if (-not $current) {
            Add-CheckResult -Category 'Databases' -Status 'FAIL' `
                -Detail "Database missing: $($db.name)"
        }
        elseif ($current.state_desc -ne $db.state_desc) {
            $status = if ($current.state_desc -ne 'ONLINE') { 'FAIL' } else { 'WARN' }
            Add-CheckResult -Category 'Databases' -Status $status `
                -Detail "$($db.name): was $($db.state_desc), now $($current.state_desc)"
        }
        elseif ($current.recovery_model_desc -ne $db.recovery_model_desc) {
            Add-CheckResult -Category 'Databases' -Status 'WARN' `
                -Detail "$($db.name): recovery model changed from $($db.recovery_model_desc) to $($current.recovery_model_desc)"
        }
        else {
            Add-CheckResult -Category 'Databases' -Status 'PASS' `
                -Detail "$($db.name) is $($current.state_desc)"
        }
    }

    /* ── Availability Groups ── */
    if ($baseline.AvailabilityGroups.Count -gt 0) {
        try {
            $currentAg = Invoke-DbaQuery -SqlInstance $SqlInstance -Query @"
                SELECT
                    ag.[name] AS [AgName]
                   ,r.[replica_server_name]
                   ,rs.[role_desc]
                   ,rs.[connected_state_desc]
                   ,rs.[synchronization_health_desc]
                   ,drs.[database_name]
                   ,drs.[synchronization_state_desc]
                   ,drs.[is_joined]
                FROM sys.[availability_groups] AS ag
                    INNER JOIN sys.[availability_replicas] AS r
                        ON ag.[group_id] = r.[group_id]
                    INNER JOIN sys.dm_hadr_availability_replica_states AS rs
                        ON r.[replica_id] = rs.[replica_id]
                    LEFT JOIN sys.dm_hadr_database_replica_states AS drs
                        ON drs.[replica_id] = r.[replica_id]
                ORDER BY ag.[name], r.[replica_server_name];
"@ -EnableException

            /* Check for disconnected or unhealthy replicas */
            $disconnected = $currentAg | Where-Object {
                $_.connected_state_desc -eq 'DISCONNECTED'
            }
            if ($disconnected) {
                $names = ($disconnected | ForEach-Object { $_.replica_server_name }) -join ', '
                Add-CheckResult -Category 'AG Health' -Status 'FAIL' `
                    -Detail "Disconnected replicas: $names"
            }

            $notSyncing = $currentAg | Where-Object {
                $_.synchronization_state_desc -notin 'SYNCHRONIZED', 'SYNCHRONIZING' -and
                $_.database_name -ne $null
            }
            if ($notSyncing) {
                $detail = ($notSyncing | ForEach-Object {
                    "$($_.database_name) on $($_.replica_server_name): $($_.synchronization_state_desc)"
                }) -join '; '
                Add-CheckResult -Category 'AG Health' -Status 'FAIL' `
                    -Detail "Not synchronizing: $detail"
            }

            /* Informational: role changes */
            foreach ($prev in ($baseline.AvailabilityGroups | Select-Object -Property replica_server_name, role_desc -Unique)) {
                $curr = $currentAg | Where-Object { $_.replica_server_name -eq $prev.replica_server_name } |
                    Select-Object -First 1
                if ($curr -and $curr.role_desc -ne $prev.role_desc) {
                    Add-CheckResult -Category 'AG Health' -Status 'INFO' `
                        -Detail "$($prev.replica_server_name): role changed from $($prev.role_desc) to $($curr.role_desc)"
                }
            }

            if (-not $disconnected -and -not $notSyncing) {
                Add-CheckResult -Category 'AG Health' -Status 'PASS' `
                    -Detail 'All replicas connected and synchronizing'
            }
        }
        catch {
            Add-CheckResult -Category 'AG Health' -Status 'WARN' `
                -Detail "Could not query AG state: $($_.Exception.Message)"
        }
    }

    /* ── Agent Jobs ── */
    $currentJobs = Get-DbaAgentJob -SqlInstance $SqlInstance -EnableException
    foreach ($job in $baseline.AgentJobs) {
        $current = $currentJobs | Where-Object { $_.Name -eq $job.Name }
        if (-not $current) {
            Add-CheckResult -Category 'Agent Jobs' -Status 'WARN' `
                -Detail "Job missing: $($job.Name)"
        }
        elseif ($job.IsEnabled -and -not $current.IsEnabled) {
            Add-CheckResult -Category 'Agent Jobs' -Status 'WARN' `
                -Detail "Job disabled after patch: $($job.Name)"
        }
    }
    $jobIssues = $checkResults | Where-Object {
        $_.Category -eq 'Agent Jobs' -and $_.Status -ne 'PASS'
    }
    if (-not $jobIssues) {
        Add-CheckResult -Category 'Agent Jobs' -Status 'PASS' `
            -Detail "All $($baseline.AgentJobs.Count) jobs match baseline"
    }

    /* ── Linked Servers ── */
    $currentLinked = Get-DbaLinkedServer -SqlInstance $SqlInstance -EnableException
    foreach ($ls in $baseline.LinkedServers) {
        $current = $currentLinked | Where-Object { $_.Name -eq $ls.Name }
        if (-not $current) {
            Add-CheckResult -Category 'Linked Servers' -Status 'WARN' `
                -Detail "Linked server missing: $($ls.Name)"
        }
    }
    $lsIssues = $checkResults | Where-Object {
        $_.Category -eq 'Linked Servers' -and $_.Status -ne 'PASS'
    }
    if (-not $lsIssues) {
        Add-CheckResult -Category 'Linked Servers' -Status 'PASS' `
            -Detail "All $($baseline.LinkedServers.Count) linked servers present"
    }

    /* ── Logins ── */
    $currentLogins = Invoke-DbaQuery -SqlInstance $SqlInstance -Query @"
        SELECT [name], [type_desc], [is_disabled]
        FROM sys.[server_principals]
        WHERE [type] IN ('S', 'U', 'G')
            AND [name] NOT LIKE N'##%'
        ORDER BY [name];
"@ -EnableException

    foreach ($login in $baseline.Logins) {
        $current = $currentLogins | Where-Object { $_.name -eq $login.name }
        if (-not $current) {
            Add-CheckResult -Category 'Logins' -Status 'WARN' `
                -Detail "Login missing: $($login.name)"
        }
        elseif ($login.is_disabled -ne $current.is_disabled) {
            Add-CheckResult -Category 'Logins' -Status 'WARN' `
                -Detail "$($login.name): disabled state changed"
        }
    }
    $loginIssues = $checkResults | Where-Object {
        $_.Category -eq 'Logins' -and $_.Status -ne 'PASS'
    }
    if (-not $loginIssues) {
        Add-CheckResult -Category 'Logins' -Status 'PASS' `
            -Detail "All $($baseline.Logins.Count) logins match"
    }

    /* ── sp_configure ── */
    $currentConfig = Get-DbaSpConfigure -SqlInstance $SqlInstance -EnableException |
        Where-Object { $_.RunningValue -ne $_.DefaultValue }
    foreach ($cfg in $baseline.SpConfigure) {
        $current = $currentConfig |
            Where-Object { $_.DisplayName -eq $cfg.DisplayName }
        if ($current -and $current.RunningValue -ne $cfg.RunningValue) {
            Add-CheckResult -Category 'Configuration' -Status 'WARN' `
                -Detail "$($cfg.DisplayName): was $($cfg.RunningValue), now $($current.RunningValue)"
        }
    }
    $cfgIssues = $checkResults | Where-Object {
        $_.Category -eq 'Configuration' -and $_.Status -ne 'PASS'
    }
    if (-not $cfgIssues) {
        Add-CheckResult -Category 'Configuration' -Status 'PASS' `
            -Detail 'sp_configure values unchanged'
    }

    /* ── Smoke Tests ── */
    if ($SmokeTestQueries) {
        foreach ($dbName in $SmokeTestQueries.Keys) {
            $query = $SmokeTestQueries[$dbName]
            try {
                $result = Invoke-DbaQuery -SqlInstance $SqlInstance `
                    -Database $dbName -Query $query -EnableException
                if ($result) {
                    Add-CheckResult -Category 'Smoke Test' -Status 'PASS' `
                        -Detail "$dbName — query returned $($result.Count) rows"
                }
                else {
                    Add-CheckResult -Category 'Smoke Test' -Status 'WARN' `
                        -Detail "$dbName — query returned no rows"
                }
            }
            catch {
                Add-CheckResult -Category 'Smoke Test' -Status 'FAIL' `
                    -Detail "$dbName — $($_.Exception.Message)"
            }
        }
    }

    /* ── Summary ── */
    Write-Host "`n=== Patch Validation Summary for $SqlInstance ===" -ForegroundColor Cyan
    Write-Host "Baseline from: $($baseline.CaptureTime)"
    Write-Host "Validated at:  $(Get-Date -Format 'yyyy-MM-dd HH:mm:ss')`n"

    $grouped = $checkResults | Group-Object -Property Status
    foreach ($group in $grouped | Sort-Object -Property Name) {
        $color = switch ($group.Name) {
            'PASS' { 'Green' }
            'FAIL' { 'Red' }
            'WARN' { 'Yellow' }
            'INFO' { 'Cyan' }
        }
        foreach ($item in $group.Group) {
            Write-Host "  [$($item.Status)] $($item.Category): $($item.Detail)" `
                -ForegroundColor $color
        }
    }

    $failCount = ($checkResults | Where-Object { $_.Status -eq 'FAIL' }).Count
    $warnCount = ($checkResults | Where-Object { $_.Status -eq 'WARN' }).Count
    $passCount = ($checkResults | Where-Object { $_.Status -eq 'PASS' }).Count

    Write-Host "`nResults: $passCount passed, $warnCount warnings, $failCount failures" `
        -ForegroundColor $(if ($failCount -gt 0) { 'Red' } else { 'Green' })

    if ($criticalFailure) {
        Write-Error "Critical failures detected — review before proceeding"
        exit 1
    }
}
```

## How I Use This

The workflow during a patching window looks like this:

```powershell
/* Pre-patch — capture baseline */
$baselineFile = Save-PatchBaseline -SqlInstance 'SQL-PROD-01' `
    -OutputPath '\\dba-share\patching\baselines' -Verbose

/* ... apply the CU, restart services ... */

/* Post-patch — validate */
$smokeTests = @{
    'AppDatabase'  = 'SELECT TOP (1) 1 FROM [dbo].[Orders];'
    'ReportingDW'  = 'SELECT TOP (1) 1 FROM [dbo].[FactSales];'
}

$splatValidation = @{
    SqlInstance      = 'SQL-PROD-01'
    BaselinePath     = $baselineFile
    ExpectedBuild    = '16.0.4175.1'
    SmokeTestQueries = $smokeTests
}
Test-PatchBaseline @splatValidation -Verbose
```

The smoke test queries are simple — just verify you can connect, read from a critical table, and get results. If the query fails, something is fundamentally wrong (permissions changed, database not online, corruption). If it succeeds, the engine is functional at a basic level.

## What I Validated

I tested this against a non-production instance by capturing a baseline, intentionally disabling a SQL Agent job and stopping the Full-Text service, then running the validation:

1. **The disabled job was caught** as a warning — exactly right, since a disabled job isn't a critical failure but needs investigation.
2. **The stopped service was caught** as a critical failure — correct, since a service that was running pre-patch should be running post-patch.
3. **AG role swaps showed as INFO** rather than FAIL — this was the iteration that mattered most. After patching a secondary and restarting, the AG might have failed over. That's not a problem; it's expected. But a disconnected replica is a problem.

For more on migration and patching workflows, see [Post 11: Migration Planning and Compatibility](/ai-for-dbas/migration-planning-compatibility/). For the PowerShell patterns used throughout, see [Post 6: PowerShell Automation](/ai-for-dbas/powershell-automation/).

## Try This Yourself

Run `Save-PatchBaseline` against any instance — you don't need to be patching. The baseline itself is useful as documentation. How many non-default `sp_configure` values does your production instance have? Are there logins that should be disabled? Linked servers you forgot about?

Then plan your next patching window. Capture baselines before patching, apply the CU, and run validation after. The first time you use this, you'll probably find something in the baseline you didn't know about — a service set to Manual start that happens to be running, or a Resource Governor pool that nobody remembers creating. That's the value: not just validating the patch, but understanding your instance's actual configuration.

---

*Part of the [ALTER DBA ADD AGENT](/ai-for-dbas/alter-dba-add-agent/) series.*
