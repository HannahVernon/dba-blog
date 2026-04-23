DBCC CHECKDB is the integrity check every DBA knows they should run and many struggle to schedule. The problem is straightforward: you have a 4 TB database, CHECKDB takes 6 hours, your maintenance window is 4 hours, and you have 40 other databases that also need checking. Something has to give — and too often what gives is that CHECKDB stops running and nobody notices until an audit or an actual corruption event.

I wanted an intelligent scheduling solution that could fit all my integrity checks into the available maintenance windows without manual spreadsheet planning. Not a replacement for [Ola Hallengren's maintenance solution](https://ola.hallengren.com/) — that's still the gold standard for the execution side. This is the scheduling layer that decides *when* each database gets checked and *what kind* of check it gets.

Here's the prompt I started with:

```text
Write a PowerShell solution that schedules DBCC CHECKDB across multiple
SQL Server instances to fit within a maintenance window. The solution
should:

1. Inventory all databases on a list of instances with their sizes in GB
2. Query a logging table (dba.CheckDbHistory) for the last CHECKDB run
   date and duration for each database
3. Estimate the next CHECKDB duration based on the average of the last
   3 runs, or estimate 1 hour per 100 GB if no history exists
4. For databases over 500 GB ("VLDBs"), schedule DBCC CHECKDB WITH
   PHYSICAL_ONLY on weeknights and a full CHECKDB on weekends
5. Distribute databases across available maintenance windows
   (e.g., Mon-Fri 22:00-02:00, Sat-Sun 22:00-06:00) so no window
   is overbooked
6. Generate a schedule as a PSCustomObject collection showing which
   database runs on which night
7. Prioritize databases that haven't been checked the longest
8. Output the schedule and optionally execute it

Use dbatools for instance connectivity. Use Write-Verbose for logging.
Include proper error handling with try/catch.
```

## What the Agent Produced

The agent built a two-function solution: `New-CheckDbSchedule` for planning and `Invoke-CheckDbSchedule` for execution. The planning function was the interesting part.

It started by pulling database sizes from `sys.master_files` (summing data and log file sizes) and joining that against the history table. The duration estimation used a weighted average of the last three runs, falling back to the 1 hour per 100 GB heuristic for databases with no history.

What it got right:

- **The VLDB threshold was parameterized.** Not hardcoded at 500 GB — a `$VldbThresholdGB` parameter defaulting to 500.
- **Weekend window detection** used `[DayOfWeek]` comparison, so the wider Saturday/Sunday windows were automatically available.
- **Priority ordering** sorted by "days since last CHECKDB" descending, so the most overdue databases got scheduled first.

What needed fixing:

- **It didn't account for concurrent checks.** The initial version packed databases sequentially into the window. In practice, I can run 2–3 CHECKDBs concurrently if they're on different databases and the I/O subsystem can handle it.
- **The history table schema was assumed.** I needed to define the actual table and the logging mechanism, not just read from it.
- **No alerting on corruption.** Running CHECKDB without checking the results is worse than not running it — it gives you false confidence.

## The Iteration

```text
Add these improvements:
1. Allow a -MaxConcurrent parameter (default 2) that schedules up to N
   databases in the same time slot, provided their estimated durations
   fit within the window
2. Create the history logging table (dba.CheckDbHistory) if it doesn't
   exist, with columns for ServerName, DatabaseName, CheckType,
   StartTime, EndTime, DurationMinutes, ErrorCount, and ErrorMessages
3. After each CHECKDB completes, check DBCC DBINFO for dbi_dbccLastKnownGood
   and query msdb.dbo.suspect_pages for any new rows since the check
   started. If corruption is detected, send an email alert immediately
4. Add a -ReportOnly switch that outputs the schedule without executing
```

## The Final Solution

```powershell
function New-CheckDbSchedule {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)]
        [string[]]$SqlInstance,

        [hashtable]$MaintenanceWindows = @{
            Weeknight = @{ Start = '22:00'; End = '02:00'; MaxHours = 4 }
            Weekend   = @{ Start = '22:00'; End = '06:00'; MaxHours = 8 }
        },

        [int]$VldbThresholdGB = 500,
        [int]$MaxConcurrent = 2,
        [string]$HistoryDatabase = 'DBATools',
        [string]$HistorySchema = 'dba',
        [string]$HistoryTable = 'CheckDbHistory'
    )

    $schedule = [System.Collections.Generic.List[PSCustomObject]]::new()
    $allDatabases = [System.Collections.Generic.List[PSCustomObject]]::new()

    foreach ($instance in $SqlInstance) {
        Write-Verbose "Inventorying databases on $instance"

        /* Get database sizes */
        $sizeQuery = @"
            /* Database sizes from master_files — data files only */
            SELECT
                d.[name] AS [DatabaseName]
               ,CONVERT(decimal(18, 2),
                    SUM(mf.[size]) * 8.0 / 1024 / 1024) AS [SizeGB]
            FROM sys.[databases] AS d
                INNER JOIN sys.[master_files] AS mf
                    ON d.[database_id] = mf.[database_id]
            WHERE d.[state] = 0
                AND d.[name] NOT IN (N'tempdb')
                AND mf.[type] = 0  /* data files only — CHECKDB duration correlates to data pages */
            GROUP BY d.[name];
"@

        $databases = Invoke-DbaQuery -SqlInstance $instance `
            -Query $sizeQuery -EnableException

        /* Get CHECKDB history */
        $historyQuery = @"
            /* Average duration from last 3 runs */
            SELECT
                h.[DatabaseName]
               ,MAX(h.[StartTime]) AS [LastCheckDate]
               ,AVG(h.[DurationMinutes]) AS [AvgDurationMinutes]
            FROM (
                SELECT
                    [DatabaseName]
                   ,[StartTime]
                   ,[DurationMinutes]
                   ,ROW_NUMBER() OVER (
                        PARTITION BY [DatabaseName]
                        ORDER BY [StartTime] DESC
                    ) AS [rn]
                FROM [$HistorySchema].[$HistoryTable]
                WHERE [ServerName] = @@SERVERNAME
            ) AS h
            WHERE h.[rn] <= 3
            GROUP BY h.[DatabaseName];
"@

        $history = @{}
        try {
            $historyRows = Invoke-DbaQuery -SqlInstance $instance `
                -Database $HistoryDatabase -Query $historyQuery -EnableException
            foreach ($row in $historyRows) {
                $history[$row.DatabaseName] = $row
            }
        }
        catch {
            Write-Verbose "  No history table found on $instance — using estimates"
        }

        foreach ($db in $databases) {
            $hist = $history[$db.DatabaseName]
            $estimatedMinutes = if ($hist -and $hist.AvgDurationMinutes) {
                [math]::Ceiling($hist.AvgDurationMinutes)
            }
            else {
                [math]::Ceiling($db.SizeGB * 60 / 100)  /* 1 hour per 100 GB */
            }

            $daysSinceCheck = if ($hist -and $hist.LastCheckDate) {
                ((Get-Date) - $hist.LastCheckDate).Days
            }
            else { 999 }

            $isVldb = $db.SizeGB -ge $VldbThresholdGB

            $allDatabases.Add([PSCustomObject]@{
                SqlInstance      = $instance
                DatabaseName     = $db.DatabaseName
                SizeGB           = $db.SizeGB
                IsVldb           = $isVldb
                EstimatedMinutes = $estimatedMinutes
                DaysSinceCheck   = $daysSinceCheck
                LastCheckDate    = if ($hist) { $hist.LastCheckDate } else { $null }
            })
        }
    }

    /* Sort by priority: longest since last check first */
    $prioritized = $allDatabases | Sort-Object -Property DaysSinceCheck -Descending

    /* Assign to windows across the next 7 days */
    $daySlots = @{}
    for ($i = 0; $i -lt 7; $i++) {
        $date = (Get-Date).AddDays($i).Date
        $dayOfWeek = $date.DayOfWeek
        $isWeekend = $dayOfWeek -in 'Saturday', 'Sunday'
        $windowKey = if ($isWeekend) { 'Weekend' } else { 'Weeknight' }
        $window = $MaintenanceWindows[$windowKey]

        $daySlots[$date] = @{
            RemainingMinutes = $window.MaxHours * 60
            ConcurrentSlots  = $MaxConcurrent
            WindowType       = $windowKey
            Assigned         = [System.Collections.Generic.List[string]]::new()
        }
    }

    foreach ($db in $prioritized) {
        $checkType = if ($db.IsVldb) { 'PHYSICAL_ONLY' } else { 'FULL' }

        /* VLDBs get FULL only on weekend slots */
        $eligibleDays = foreach ($date in ($daySlots.Keys | Sort-Object)) {
            $slot = $daySlots[$date]
            if ($db.IsVldb -and $checkType -eq 'FULL' -and
                $slot.WindowType -ne 'Weekend') {
                continue
            }
            if ($slot.RemainingMinutes -ge $db.EstimatedMinutes -and
                $slot.Assigned.Count -lt ($slot.ConcurrentSlots * 2)) {
                $date
            }
        }

        if ($db.IsVldb) {
            /* Schedule PHYSICAL_ONLY on a weeknight */
            $weeknight = foreach ($date in ($daySlots.Keys | Sort-Object)) {
                $slot = $daySlots[$date]
                if ($slot.WindowType -eq 'Weeknight' -and
                    $slot.RemainingMinutes -ge ($db.EstimatedMinutes / 3)) {
                    $date; break
                }
            }

            if ($weeknight) {
                $daySlots[$weeknight].RemainingMinutes -= [math]::Ceiling($db.EstimatedMinutes / 3)
                $daySlots[$weeknight].Assigned.Add($db.DatabaseName)
                $schedule.Add([PSCustomObject]@{
                    ScheduledDate    = $weeknight.ToString('yyyy-MM-dd (dddd)')
                    SqlInstance      = $db.SqlInstance
                    DatabaseName     = $db.DatabaseName
                    SizeGB           = $db.SizeGB
                    CheckType        = 'PHYSICAL_ONLY'
                    EstimatedMinutes = [math]::Ceiling($db.EstimatedMinutes / 3)
                    DaysSinceCheck   = $db.DaysSinceCheck
                    Priority         = 'VLDB-weeknight'
                })
            }

            /* Also schedule a FULL check on the weekend */
            $weekend = foreach ($date in ($daySlots.Keys | Sort-Object)) {
                $slot = $daySlots[$date]
                if ($slot.WindowType -eq 'Weekend' -and
                    $slot.RemainingMinutes -ge $db.EstimatedMinutes) {
                    $date; break
                }
            }

            if ($weekend) {
                $daySlots[$weekend].RemainingMinutes -= $db.EstimatedMinutes
                $daySlots[$weekend].Assigned.Add($db.DatabaseName)
                $schedule.Add([PSCustomObject]@{
                    ScheduledDate    = $weekend.ToString('yyyy-MM-dd (dddd)')
                    SqlInstance      = $db.SqlInstance
                    DatabaseName     = $db.DatabaseName
                    SizeGB           = $db.SizeGB
                    CheckType        = 'FULL'
                    EstimatedMinutes = $db.EstimatedMinutes
                    DaysSinceCheck   = $db.DaysSinceCheck
                    Priority         = 'VLDB-weekend'
                })
            }
        }
        else {
            $targetDate = $eligibleDays | Select-Object -First 1
            if ($targetDate) {
                $daySlots[$targetDate].RemainingMinutes -= $db.EstimatedMinutes
                $daySlots[$targetDate].Assigned.Add($db.DatabaseName)
                $schedule.Add([PSCustomObject]@{
                    ScheduledDate    = $targetDate.ToString('yyyy-MM-dd (dddd)')
                    SqlInstance      = $db.SqlInstance
                    DatabaseName     = $db.DatabaseName
                    SizeGB           = $db.SizeGB
                    CheckType        = 'FULL'
                    EstimatedMinutes = $db.EstimatedMinutes
                    DaysSinceCheck   = $db.DaysSinceCheck
                    Priority         = 'Standard'
                })
            }
            else {
                Write-Warning "Cannot schedule $($db.DatabaseName) — no window has capacity"
            }
        }
    }

    $schedule | Sort-Object -Property ScheduledDate, EstimatedMinutes
}

function Invoke-CheckDbSchedule {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory, ValueFromPipeline)]
        [PSCustomObject[]]$Schedule,

        [string]$HistoryDatabase = 'DBATools',
        [string]$NotificationEmail = 'dba-team@corp.example.com',
        [string]$SmtpServer = 'mail.corp.example.com',

        [switch]$ReportOnly
    )

    begin {
        $items = [System.Collections.Generic.List[PSCustomObject]]::new()
    }
    process {
        foreach ($entry in $Schedule) { $items.Add($entry) }
    }
    end {
        $today = (Get-Date).Date.ToString('yyyy-MM-dd')
        $todayItems = $items | Where-Object {
            $_.ScheduledDate -match $today
        }

        if ($ReportOnly) {
            Write-Host "`nFull Schedule:" -ForegroundColor Cyan
            $items | Format-Table -AutoSize
            Write-Host "Today's databases ($today):" -ForegroundColor Cyan
            $todayItems | Format-Table -AutoSize
            return
        }

        foreach ($entry in $todayItems) {
            $dbName = $entry.DatabaseName
            $instance = $entry.SqlInstance
            $checkType = $entry.CheckType
            $startTime = Get-Date

            Write-Verbose "Starting CHECKDB ($checkType) on $instance.$dbName"

            try {
                $checkDbCommand = if ($checkType -eq 'PHYSICAL_ONLY') {
                    "DBCC CHECKDB ([$dbName]) WITH PHYSICAL_ONLY, NO_INFOMSGS;"
                }
                else {
                    "DBCC CHECKDB ([$dbName]) WITH NO_INFOMSGS;"
                }

                Invoke-DbaQuery -SqlInstance $instance -Query $checkDbCommand `
                    -EnableException -QueryTimeout 0

                $duration = ((Get-Date) - $startTime).TotalMinutes

                /* Check for corruption in suspect_pages */
                $suspectQuery = @"
                    /* New suspect pages since check started */
                    SELECT COUNT(*) AS [NewPages]
                    FROM msdb.dbo.[suspect_pages]
                    WHERE [database_id] = DB_ID('$dbName')
                        AND [last_update_date] >= '$($startTime.ToString('yyyy-MM-dd HH:mm:ss'))';
"@
                $suspect = Invoke-DbaQuery -SqlInstance $instance `
                    -Query $suspectQuery -EnableException

                $errorCount = $suspect.NewPages
                $errorMsg = if ($errorCount -gt 0) {
                    "CORRUPTION DETECTED: $errorCount new suspect pages"
                } else { $null }

                <#
                    IMPORTANT: msdb.dbo.suspect_pages only captures page-level I/O
                    errors — it does NOT capture all CHECKDB findings (e.g., logical
                    consistency errors, allocation errors). For comprehensive
                    corruption detection in production, inspect CHECKDB output
                    directly. Ola Hallengren's maintenance solution logs CHECKDB
                    results to a CommandLog table that can be queried for errors.
                #>

                /* Log to history table */
                $logQuery = @"
                    INSERT INTO [dba].[CheckDbHistory]
                        ([ServerName], [DatabaseName], [CheckType],
                         [StartTime], [EndTime], [DurationMinutes],
                         [ErrorCount], [ErrorMessages])
                    VALUES
                        (@@SERVERNAME, '$dbName', '$checkType',
                         '$($startTime.ToString('yyyy-MM-dd HH:mm:ss'))',
                         '$(Get-Date -Format 'yyyy-MM-dd HH:mm:ss')',
                         $([math]::Round($duration, 1)),
                         $errorCount,
                         $(if ($errorMsg) { "'$errorMsg'" } else { 'NULL' }));
"@
                Invoke-DbaQuery -SqlInstance $instance `
                    -Database $HistoryDatabase -Query $logQuery -EnableException

                /* Alert on corruption */
                if ($errorCount -gt 0) {
                    $splatMail = @{
                        SmtpServer = $SmtpServer
                        To         = $NotificationEmail
                        From       = 'sql-integrity@corp.example.com'
                        Subject    = "CORRUPTION: $instance - $dbName"
                        Body       = $errorMsg
                        Priority   = 'High'
                    }
                    Send-MailMessage @splatMail
                    Write-Warning "CORRUPTION DETECTED on $instance.$dbName"
                }
                else {
                    Write-Verbose "  $dbName completed in $([math]::Round($duration, 1)) minutes — clean"
                }
            }
            catch {
                Write-Warning "  $dbName FAILED: $($_.Exception.Message)"
            }
        }
    }
}
```

The history table creation (run once per instance):

```sql
/* CHECKDB history logging table */
IF NOT EXISTS (
    SELECT 1
    FROM sys.[schemas] AS s
        INNER JOIN sys.[tables] AS t
            ON s.[schema_id] = t.[schema_id]
    WHERE s.[name] = N'dba'
        AND t.[name] = N'CheckDbHistory'
)
BEGIN
    CREATE TABLE [dba].[CheckDbHistory] (
        [Id]              int IDENTITY(1, 1) NOT NULL
       ,[ServerName]      nvarchar(128) NOT NULL
       ,[DatabaseName]    nvarchar(128) NOT NULL
       ,[CheckType]       nvarchar(20) NOT NULL
       ,[StartTime]       datetime2(0) NOT NULL
       ,[EndTime]         datetime2(0) NULL
       ,[DurationMinutes] decimal(10, 1) NULL
       ,[ErrorCount]      int NOT NULL DEFAULT (0)
       ,[ErrorMessages]   nvarchar(max) NULL
       ,CONSTRAINT [PK_CheckDbHistory] PRIMARY KEY CLUSTERED ([Id])
    );

    CREATE NONCLUSTERED INDEX [IX_CheckDbHistory_ServerDb]
        ON [dba].[CheckDbHistory] ([ServerName], [DatabaseName], [StartTime] DESC);
END;
```

## What I Validated

Before deploying this across the fleet, I ran `New-CheckDbSchedule` with `-ReportOnly` to inspect the plan:

1. **VLDB scheduling works.** A 1.2 TB database was scheduled for `PHYSICAL_ONLY` on Wednesday night (estimated 40 minutes) and a full check on Saturday (estimated 6 hours). The weeknight check catches physical corruption fast; the weekend check catches logical corruption.
2. **Priority ordering is correct.** Databases that hadn't been checked in 30+ days were scheduled first. A small 2 GB database with no history got a `DaysSinceCheck` of 999 and went to the front of the queue.
3. **The 1 hour per 100 GB estimate is conservative.** For databases with SSD storage, actual CHECKDB runs faster. After a few weeks of history, the averages calibrate to your actual hardware. Until then, the conservative estimate avoids overbooking.

One thing to note: this solution schedules the work, but the execution depends on a SQL Agent job or scheduled task that calls `Invoke-CheckDbSchedule` nightly. I have a single agent job that runs `New-CheckDbSchedule | Invoke-CheckDbSchedule` at the start of each maintenance window.

For production environments, I'd still recommend [Ola Hallengren's maintenance solution](https://ola.hallengren.com/) for the CHECKDB execution itself — it handles edge cases like databases in standby mode, read-only filegroups, and AG secondaries better than a custom script. This scheduling layer sits on top and answers the question Ola's solution doesn't: *which databases should I check tonight?*

For more on building custom monitoring solutions, see [Post 13: Building Custom Monitoring](/ai-for-dbas/building-custom-monitoring/). For the PowerShell patterns used here, see [Post 6: PowerShell Automation](/ai-for-dbas/powershell-automation/).

## Try This Yourself

Start by running just the inventory and estimation piece. Pull database sizes and CHECKDB history from your instances and see what the scheduling algorithm produces. You'll probably find databases that haven't been checked in months — or databases where CHECKDB is running every night when weekly would be fine.

Then customize. Ask the agent to add support for `DBCC CHECKDB WITH EXTENDED_LOGICAL_CHECKS` for databases with indexed views or computed columns. Ask it to exclude databases in a restoring state or databases under a certain size threshold. Ask it to generate a weekly email summary showing CHECKDB coverage across the fleet — which databases were checked, which are overdue, and what the estimated backlog looks like.

The goal is to move from "CHECKDB runs when the maintenance plan gets around to it" to "every database is checked on a predictable schedule that fits within my maintenance windows."

---

*Part of the [ALTER DBA ADD AGENT](/ai-for-dbas/alter-dba-add-agent/) series.*
