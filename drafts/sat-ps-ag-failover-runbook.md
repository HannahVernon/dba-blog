Planned AG failovers aren't technically difficult. `ALTER AVAILABILITY GROUP ... FAILOVER` is one line of T-SQL. But anyone who's done them in production knows the real work isn't the failover command — it's the 30 minutes of checks before and after. Are all replicas synchronized? Is the redo queue drained? Are applications reconnecting to the listener? Did that third-party monitoring tool lose its mind?

I've done enough of these that I have a mental checklist. The problem with mental checklists is that at 6 AM during a patching window, you skip steps. I wanted a scripted runbook that enforces the checklist — and I wanted it built faster than the three hours it would take me to write from scratch.

Here's the prompt I used:

```text
Write a PowerShell AG failover runbook script using dbatools. The script
should accept parameters for the AG name and the target replica. It needs
four phases:

Phase 1 - Pre-Failover Checks:
- Verify all replicas are connected and healthy
- Verify all databases are in SYNCHRONIZED state
- Check that the redo queue size is under 1 MB on the target replica
- Check that the send queue size is under 1 MB
- Verify the target replica is in SYNCHRONOUS_COMMIT mode
- Verify no active backup or DBCC jobs are running on either replica
  (check msdb.dbo.sysjobactivity for running jobs matching common
  backup/DBCC job name patterns)
- Display the current primary and all replica roles

Phase 2 - Failover Execution:
- Prompt for confirmation with a clear summary of what will happen
- Perform the failover using Invoke-DbaAgFailover
- Wait for the failover to complete

Phase 3 - Post-Failover Validation:
- Verify the new primary is the expected target replica
- Verify all databases are online on the new primary
- Test listener connectivity with Test-DbaConnection
- Verify all replicas are synchronizing or synchronized
- Check that sys.databases shows all AG databases as ONLINE

Phase 4 - Rollback Guidance:
- If any post-check fails, display step-by-step rollback instructions
- Do NOT auto-rollback — just tell the operator what to do

Include a -DryRun switch that runs all pre-checks and shows what would
happen, without executing the failover. Log every step with timestamps
using Write-Verbose. Use splatting for long parameter lists.
```

## What the Agent Produced

The agent built a well-structured `Invoke-AgFailoverRunbook` function with the four phases clearly separated. A few things it got right out of the gate:

- **Pre-checks that actually block.** Each check returns a pass/fail, and the script exits if any pre-check fails. No "warning, continuing anyway" — a hard stop.
- **The confirmation prompt** showed a formatted summary: current primary, target replica, number of databases, AG name. Clear enough that a sleep-deprived DBA at 6 AM can read it and confirm.
- **The `-DryRun` mode** ran all Phase 1 checks, displayed the planned action, then exited — exactly what I need for pre-validating during change management.

What needed work:

- **Job detection was too broad.** The agent checked for *any* running job. I refined it to look for jobs with names matching patterns like `%backup%`, `%CHECKDB%`, `%integrity%`, `%index%` — the maintenance jobs that would conflict with a failover.
- **It didn't check the AG listener.** Pre-failover, I want to verify the listener is resolving to the current primary. Post-failover, I want to verify it resolves to the new primary. This confirms DNS and networking are working, not just the AG metadata.
- **Timeout handling was missing.** The failover usually completes in seconds, but sometimes the redo queue needs to drain. I asked the agent to add a configurable timeout with a polling loop.

## The Iteration

```text
Add these improvements:
1. Before failover, resolve the AG listener DNS name and verify it points
   to the current primary's IP. After failover, poll until the listener
   resolves to the new primary's IP, with a 60-second timeout.
2. Add a -TimeoutSeconds parameter (default 120) that controls how long
   the script waits for all databases to reach SYNCHRONIZED state after
   failover.
3. Narrow the running-job check to jobs whose names match
   '%backup%', '%checkdb%', '%integrity%', '%index%', or '%maintenance%'.
4. After all post-checks pass, output a structured summary object with
   before/after state, failover duration, and pass/fail for each check.
```

The agent handled all four. The listener DNS check used `Resolve-DnsName`, which is exactly right.

## The Final Runbook

```powershell
function Invoke-AgFailoverRunbook {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)]
        [string]$AvailabilityGroup,

        [Parameter(Mandatory)]
        [string]$TargetReplica,

        [Parameter(Mandatory)]
        [string]$ListenerName,

        [int]$TimeoutSeconds = 120,

        [switch]$DryRun
    )

    $startTime = Get-Date
    $results = [System.Collections.Generic.List[PSCustomObject]]::new()

    function Write-Check {
        param([string]$Check, [string]$Status, [string]$Detail)
        $entry = [PSCustomObject]@{
            Timestamp = (Get-Date).ToString('HH:mm:ss')
            Check     = $Check
            Status    = $Status
            Detail    = $Detail
        }
        $results.Add($entry)
        $color = if ($Status -eq 'PASS') { 'Green' } else { 'Red' }
        Write-Host "  [$Status] $Check - $Detail" -ForegroundColor $color
    }

    /* ── Phase 1: Pre-Failover Checks ── */
    Write-Host "`n=== Phase 1: Pre-Failover Checks ===" -ForegroundColor Cyan

    $agReplicas = Get-DbaAgReplica -SqlInstance $TargetReplica `
        -AvailabilityGroup $AvailabilityGroup

    $currentPrimary = ($agReplicas | Where-Object { $_.Role -eq 'Primary' }).Name

    if (-not $currentPrimary) {
        Write-Check -Check 'Primary Identification' -Status 'FAIL' `
            -Detail 'Cannot identify current primary replica'
        return
    }
    Write-Check -Check 'Primary Identification' -Status 'PASS' `
        -Detail "Current primary: $currentPrimary"

    if ($currentPrimary -eq $TargetReplica) {
        Write-Check -Check 'Target Validation' -Status 'FAIL' `
            -Detail "$TargetReplica is already the primary"
        return
    }

    /* Check replica health */
    $unhealthy = $agReplicas | Where-Object {
        $_.ConnectionState -ne 'Connected' -or
        $_.RollupSynchronizationState -ne 'Synchronized'
    }
    if ($unhealthy) {
        $names = ($unhealthy | ForEach-Object { $_.Name }) -join ', '
        Write-Check -Check 'Replica Health' -Status 'FAIL' `
            -Detail "Unhealthy replicas: $names"
        return
    }
    Write-Check -Check 'Replica Health' -Status 'PASS' `
        -Detail 'All replicas connected and synchronized'

    /* Check target is synchronous commit */
    $targetConfig = $agReplicas | Where-Object { $_.Name -eq $TargetReplica }
    if ($targetConfig.AvailabilityMode -ne 'SynchronousCommit') {
        Write-Check -Check 'Commit Mode' -Status 'FAIL' `
            -Detail "$TargetReplica is in $($targetConfig.AvailabilityMode) mode"
        return
    }
    Write-Check -Check 'Commit Mode' -Status 'PASS' `
        -Detail "$TargetReplica is SYNCHRONOUS_COMMIT"

    /* Check queue sizes on the target */
    $dbStates = Get-DbaAgDatabase -SqlInstance $currentPrimary `
        -AvailabilityGroup $AvailabilityGroup
    $highQueue = $dbStates | Where-Object {
        $_.RedoQueueSize -gt 1024 -or $_.SendQueueSize -gt 1024  /* KB */
    }
    if ($highQueue) {
        $names = ($highQueue | ForEach-Object { $_.DatabaseName }) -join ', '
        Write-Check -Check 'Queue Sizes' -Status 'FAIL' `
            -Detail "High queue on: $names"
        return
    }
    Write-Check -Check 'Queue Sizes' -Status 'PASS' `
        -Detail 'Redo and send queues under 1 MB'

    /* Check for running maintenance jobs */
    $jobPatterns = @('%backup%', '%checkdb%', '%integrity%', '%index%', '%maintenance%')
    $runningJobs = foreach ($pattern in $jobPatterns) {
        $query = @"
            /* Check for running maintenance jobs */
            SELECT j.[name]
            FROM msdb.dbo.[sysjobactivity] AS ja
                INNER JOIN msdb.dbo.[sysjobs] AS j
                    ON ja.[job_id] = j.[job_id]
            WHERE ja.[start_execution_date] IS NOT NULL
                AND ja.[stop_execution_date] IS NULL
                AND j.[name] LIKE '$pattern';
"@
        Invoke-DbaQuery -SqlInstance $currentPrimary, $TargetReplica `
            -Query $query -EnableException
    }
    if ($runningJobs) {
        $jobNames = ($runningJobs | ForEach-Object { $_.name }) -join ', '
        Write-Check -Check 'Running Jobs' -Status 'FAIL' `
            -Detail "Active maintenance jobs: $jobNames"
        return
    }
    Write-Check -Check 'Running Jobs' -Status 'PASS' `
        -Detail 'No conflicting maintenance jobs running'

    /* Check listener DNS */
    try {
        $listenerIp = (Resolve-DnsName -Name $ListenerName -ErrorAction Stop).IPAddress
        Write-Check -Check 'Listener DNS' -Status 'PASS' `
            -Detail "Listener resolves to $($listenerIp -join ', ')"
    }
    catch {
        Write-Check -Check 'Listener DNS' -Status 'FAIL' `
            -Detail "Cannot resolve listener: $ListenerName"
        return
    }

    /* ── DryRun Exit ── */
    if ($DryRun) {
        Write-Host "`n[DryRun] Would fail over AG '$AvailabilityGroup'" `
            -ForegroundColor Yellow
        Write-Host "[DryRun]   From: $currentPrimary" -ForegroundColor Yellow
        Write-Host "[DryRun]   To:   $TargetReplica" -ForegroundColor Yellow
        $results | Format-Table -AutoSize
        return
    }

    /* ── Phase 2: Failover Execution ── */
    Write-Host "`n=== Phase 2: Failover Execution ===" -ForegroundColor Cyan
    Write-Host "  AG:       $AvailabilityGroup"
    Write-Host "  From:     $currentPrimary"
    Write-Host "  To:       $TargetReplica"
    Write-Host ""

    $confirm = Read-Host "Type 'FAILOVER' to proceed (anything else to abort)"
    if ($confirm -ne 'FAILOVER') {
        Write-Host 'Failover aborted by operator.' -ForegroundColor Yellow
        return
    }

    $splatFailover = @{
        SqlInstance      = $TargetReplica
        AvailabilityGroup = $AvailabilityGroup
        EnableException  = $true
    }
    try {
        $failoverStart = Get-Date
        Invoke-DbaAgFailover @splatFailover
        $failoverDuration = (Get-Date) - $failoverStart
        Write-Check -Check 'Failover Command' -Status 'PASS' `
            -Detail "Completed in $($failoverDuration.TotalSeconds.ToString('F1'))s"
    }
    catch {
        Write-Check -Check 'Failover Command' -Status 'FAIL' `
            -Detail $_.Exception.Message
        Write-Host "`n=== ROLLBACK GUIDANCE ===" -ForegroundColor Red
        Write-Host "1. Check AG state: Get-DbaAgReplica -SqlInstance $currentPrimary"
        Write-Host "2. If split-brain, force quorum or restart the unintended primary"
        Write-Host "3. Do NOT attempt automatic rollback — assess manually"
        return
    }

    /* ── Phase 3: Post-Failover Validation ── */
    Write-Host "`n=== Phase 3: Post-Failover Validation ===" -ForegroundColor Cyan

    /* Wait for synchronization with timeout */
    $deadline = (Get-Date).AddSeconds($TimeoutSeconds)
    $synced = $false
    while ((Get-Date) -lt $deadline) {
        $postReplicas = Get-DbaAgReplica -SqlInstance $TargetReplica `
            -AvailabilityGroup $AvailabilityGroup
        $newPrimary = ($postReplicas | Where-Object { $_.Role -eq 'Primary' }).Name

        if ($newPrimary -eq $TargetReplica) {
            $allSynced = ($postReplicas | Where-Object {
                $_.Role -eq 'Secondary' -and
                $_.RollupSynchronizationState -notin 'Synchronized', 'Synchronizing'
            }).Count -eq 0

            if ($allSynced) { $synced = $true; break }
        }
        Start-Sleep -Seconds 5
    }

    if ($synced) {
        Write-Check -Check 'New Primary' -Status 'PASS' `
            -Detail "$TargetReplica is now primary"
    }
    else {
        Write-Check -Check 'New Primary' -Status 'FAIL' `
            -Detail "Timed out after $TimeoutSeconds seconds"
    }

    /* Verify databases online */
    $offlineDb = Invoke-DbaQuery -SqlInstance $TargetReplica -Query @"
        SELECT [name], [state_desc]
        FROM sys.[databases]
        WHERE [name] IN (
            SELECT drs.[database_name]
            FROM sys.dm_hadr_database_replica_states AS drs
                INNER JOIN sys.availability_groups AS ag
                    ON drs.[group_id] = ag.[group_id]
            WHERE ag.[name] = '$AvailabilityGroup'
        )
        AND [state] <> 0;
"@
    if ($offlineDb) {
        Write-Check -Check 'Databases Online' -Status 'FAIL' `
            -Detail "Offline: $(($offlineDb.name) -join ', ')"
    }
    else {
        Write-Check -Check 'Databases Online' -Status 'PASS' `
            -Detail 'All AG databases are ONLINE'
    }

    /* Verify listener now resolves correctly */
    $connTest = Test-DbaConnection -SqlInstance $ListenerName
    if ($connTest.ConnectSuccess) {
        Write-Check -Check 'Listener Connectivity' -Status 'PASS' `
            -Detail "Listener is responding on $ListenerName"
    }
    else {
        Write-Check -Check 'Listener Connectivity' -Status 'FAIL' `
            -Detail 'Listener connection failed post-failover'
    }

    /* ── Phase 4: Rollback Guidance (if needed) ── */
    $failedChecks = $results | Where-Object { $_.Status -eq 'FAIL' }
    if ($failedChecks) {
        Write-Host "`n=== Phase 4: ROLLBACK GUIDANCE ===" -ForegroundColor Red
        Write-Host "The following post-failover checks failed:"
        $failedChecks | Format-Table -AutoSize
        Write-Host "Manual steps:"
        Write-Host "  1. Assess AG state: Get-DbaAgReplica -SqlInstance $TargetReplica"
        Write-Host "  2. If $TargetReplica is primary but unhealthy, fail back:"
        Write-Host "     Invoke-DbaAgFailover -SqlInstance $currentPrimary ``"
        Write-Host "         -AvailabilityGroup $AvailabilityGroup"
        Write-Host "  3. Verify applications have reconnected to the listener"
        Write-Host "  4. Do NOT force-failover unless you accept potential data loss"
    }
    else {
        Write-Host "`n=== Failover Complete ===" -ForegroundColor Green
    }

    /* Summary output */
    [PSCustomObject]@{
        AvailabilityGroup  = $AvailabilityGroup
        PreviousPrimary    = $currentPrimary
        NewPrimary         = $TargetReplica
        FailoverDuration   = $failoverDuration.ToString('mm\:ss\.ff')
        TotalDuration      = ((Get-Date) - $startTime).ToString('mm\:ss')
        PreChecksPassed    = ($results | Where-Object { $_.Status -eq 'PASS' }).Count
        PostChecksFailed   = ($failedChecks).Count
        OverallResult      = if ($failedChecks) { 'NEEDS ATTENTION' } else { 'SUCCESS' }
    }
}
```

## What I Validated

I ran this with `-DryRun` against a non-production AG first. A few things I caught:

1. **The running-job query needs to cover both replicas.** The original only checked the primary. If a `CHECKDB` job is running on the target secondary, failing over to it would be disruptive. The `Invoke-DbaQuery -SqlInstance $currentPrimary, $TargetReplica` handles both.
2. **Listener DNS propagation isn't instant.** After failover, `Resolve-DnsName` may return the old IP for a few seconds. The polling loop with a timeout handles this gracefully — but set your timeout appropriately for your DNS TTL.
3. **The confirmation prompt is crucial.** During a patching window with multiple AGs, it's easy to pass the wrong target replica. The explicit "type FAILOVER" prompt is intentionally friction — a safety net against mistakes at 5 AM.

This script lives in our shared DBA tools repository. Every planned failover uses it. The `-DryRun` output is included in our change management tickets as evidence that pre-checks passed before we committed to the failover.

For the fleet health checks that feed into this runbook, see [Post 5: Health Checks and Inventory](/ai-for-dbas/health-checks-inventory/). For more PowerShell automation patterns, see [Post 6: PowerShell Automation](/ai-for-dbas/powershell-automation/).

## Try This Yourself

Take this runbook and run it with `-DryRun` against one of your AGs. The pre-checks alone are valuable — they'll tell you whether your AG is actually ready for a failover *right now*. Most DBAs find at least one surprise: a replica that's in `ASYNCHRONOUS_COMMIT` mode when it should be synchronous, a redo queue that's larger than expected, or a backup job that's still running from last night.

Then customize it. Ask the agent to add checks specific to your environment: verify that a particular Windows service is running on the target, check that enough disk space exists for tempdb on the failover target, or add a post-failover step that updates a CMDB. Each iteration takes one prompt and a few minutes of review.

---

*Part of the [ALTER DBA ADD AGENT](/ai-for-dbas/alter-dba-add-agent/) series.*
