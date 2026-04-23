If you manage more than a couple of Availability Groups, you've probably experienced the moment where someone asks "which replicas are synchronous on AG3?" and you have to connect to the primary, open the dashboard, and click around. Or worse, someone changes a replica from asynchronous to synchronous commit "for testing" and forgets to change it back.

I wanted two things: a diagnostic query that shows the complete AG topology in one result set, and a way to detect misconfigurations automatically. The AI agent built both — and the iterative process of refining the output is a good example of how these conversations work in practice.

## The First Prompt

```text
Write a T-SQL diagnostic query that maps the complete Availability Group
topology on the current instance. Show:
1. AG name and cluster type
2. Each replica: server name, role (PRIMARY/SECONDARY), availability mode
   (SYNCHRONOUS_COMMIT/ASYNCHRONOUS_COMMIT), failover mode, seeding mode
3. Synchronization health and connected state for each replica
4. Endpoint URL for each replica
5. Listener name, IP address, and port for each AG
6. Read-only routing URL and read-only routing list for each replica
7. Log send queue size and redo queue size per database per replica

Use sys.availability_groups, sys.availability_replicas,
sys.dm_hadr_availability_replica_states,
sys.dm_hadr_database_replica_states, sys.availability_group_listeners,
sys.availability_group_listener_ip_addresses,
and sys.availability_read_only_routing_lists.

Run from the primary replica. Format as multiple result sets if needed
for readability.
```

## What the Agent Produced

The agent generated three separate queries — a smart choice, because cramming all this into one result set would be unreadable:

1. **Replica overview:** AG name, replica server name, role, availability mode, failover mode, seeding mode, synchronization health, connected state, endpoint URL.
2. **Database-level detail:** AG name, database name, replica server name, synchronization state, log send queue size (KB), redo queue size (KB), last commit time.
3. **Listener and routing:** AG name, listener DNS name, IP address, port, read-only routing URL per replica, read-only routing list.

The join structure was correct — `sys.availability_replicas` joined to `sys.dm_hadr_availability_replica_states` on `replica_id`, with the database-level DMV joined on both `replica_id` and `group_database_id`. The agent correctly used `LEFT JOIN` for the listener queries since not every AG has a listener configured (distributed AGs, for example).

## The Iteration: Misconfiguration Detection

The raw topology data is useful, but I wanted the script to actively flag problems. Here's the follow-up prompt:

```text
Add a misconfiguration detection section that flags:
1. Async replicas configured with automatic failover
   (automatic failover requires synchronous commit)
2. Replicas with no read-only routing URL configured
   (means read-intent connections won't route to secondaries)
3. Synchronous commit replicas not in SYNCHRONIZED state
   (indicates a problem — latency, network issues, or suspended data movement)
4. Replicas in a DISCONNECTED state
5. Databases with redo queue > 500 MB (secondary falling behind)
6. Databases with log send queue > 500 MB (network or throughput issue)
7. Any replica with seeding_mode = MANUAL where the database is not yet joined

Label each finding with a severity: CRITICAL, WARNING, or INFO.
```

The agent produced a `UNION ALL` query that checked each condition and returned a uniform result set: `[Severity]`, `[Category]`, `[AGName]`, `[ReplicaServer]`, `[DatabaseName]`, `[Finding]`. Clean and actionable.

I refined one thing: the redo and log send queue thresholds. 500 MB is a reasonable starting point, but in environments with high transaction volumes or large databases, a sustained queue of 200 MB might already indicate a problem. I parameterized the thresholds:

```sql
DECLARE @RedoQueueThresholdKB bigint = 512000;  /* 500 MB */
DECLARE @SendQueueThresholdKB bigint = 512000;   /* 500 MB */
```

## The Final Script

```sql
/* AG Topology Diagnostic — run from primary replica */
SET NOCOUNT ON;

DECLARE @RedoQueueThresholdKB bigint = 512000;
DECLARE @SendQueueThresholdKB bigint = 512000;

/* Result Set 1: Replica Overview */
SELECT
    ag.[name] AS [AGName]
   ,ar.[replica_server_name] AS [ReplicaServer]
   ,ars.[role_desc] AS [CurrentRole]
   ,ar.[availability_mode_desc] AS [AvailabilityMode]
   ,ar.[failover_mode_desc] AS [FailoverMode]
   ,ar.[seeding_mode_desc] AS [SeedingMode]
   ,ars.[synchronization_health_desc] AS [SyncHealth]
   ,ars.[connected_state_desc] AS [ConnectedState]
   ,ar.[endpoint_url] AS [EndpointURL]
FROM sys.[availability_groups] AS ag
    INNER JOIN sys.[availability_replicas] AS ar
        ON ag.[group_id] = ar.[group_id]
    LEFT JOIN sys.[dm_hadr_availability_replica_states] AS ars
        ON ar.[replica_id] = ars.[replica_id]
ORDER BY ag.[name], ars.[role_desc] DESC, ar.[replica_server_name];

/* Result Set 2: Database-Level Synchronization */
SELECT
    ag.[name] AS [AGName]
   ,adc.[database_name] AS [DatabaseName]
   ,ar.[replica_server_name] AS [ReplicaServer]
   ,drs.[synchronization_state_desc] AS [SyncState]
   ,drs.[synchronization_health_desc] AS [SyncHealth]
   ,drs.[log_send_queue_size] AS [LogSendQueueKB]
   ,drs.[redo_queue_size] AS [RedoQueueKB]
   ,drs.[last_commit_time] AS [LastCommitTime]
   ,DATEDIFF(SECOND, drs.[last_commit_time], GETDATE())
        AS [CommitLagSeconds]
   ,drs.[is_suspended] AS [IsSuspended]
   ,drs.[suspend_reason_desc] AS [SuspendReason]
FROM sys.[availability_groups] AS ag
    INNER JOIN sys.[availability_replicas] AS ar
        ON ag.[group_id] = ar.[group_id]
    INNER JOIN sys.[dm_hadr_database_replica_states] AS drs
        ON ar.[replica_id] = drs.[replica_id]
    INNER JOIN sys.[availability_databases_cluster] AS adc
        ON drs.[group_database_id] = adc.[group_database_id]
ORDER BY ag.[name], adc.[database_name], ar.[replica_server_name];

/* Result Set 3: Listeners and Read-Only Routing */
SELECT
    ag.[name] AS [AGName]
   ,agl.[dns_name] AS [ListenerName]
   ,lip.[ip_address] AS [ListenerIP]
   ,agl.[port] AS [ListenerPort]
   ,ar.[replica_server_name] AS [ReplicaServer]
   ,ar.[read_only_routing_url] AS [ReadOnlyRoutingURL]
FROM sys.[availability_groups] AS ag
    LEFT JOIN sys.[availability_group_listeners] AS agl
        ON ag.[group_id] = agl.[group_id]
    LEFT JOIN sys.[availability_group_listener_ip_addresses] AS lip
        ON agl.[listener_id] = lip.[listener_id]
    INNER JOIN sys.[availability_replicas] AS ar
        ON ag.[group_id] = ar.[group_id]
ORDER BY ag.[name], ar.[replica_server_name];

/* Result Set 4: Misconfiguration Detection */
SELECT
    [Severity], [Category], [AGName],
    [ReplicaServer], [DatabaseName], [Finding]
FROM (
    /* Async replica with automatic failover */
    SELECT
        N'CRITICAL' AS [Severity]
       ,N'Invalid Config' AS [Category]
       ,ag.[name] AS [AGName]
       ,ar.[replica_server_name] AS [ReplicaServer]
       ,NULL AS [DatabaseName]
       ,N'Asynchronous replica configured for automatic failover — '
        + N'automatic failover requires synchronous commit' AS [Finding]
    FROM sys.[availability_groups] AS ag
        INNER JOIN sys.[availability_replicas] AS ar
            ON ag.[group_id] = ar.[group_id]
    WHERE ar.[availability_mode] = 0 /* ASYNCHRONOUS_COMMIT */
        AND ar.[failover_mode] = 1   /* AUTOMATIC */

    UNION ALL

    /* Missing read-only routing URL */
    SELECT
        N'WARNING' AS [Severity]
       ,N'Routing' AS [Category]
       ,ag.[name]
       ,ar.[replica_server_name]
       ,NULL
       ,N'No read-only routing URL configured — read-intent '
        + N'connections will not route to this secondary'
    FROM sys.[availability_groups] AS ag
        INNER JOIN sys.[availability_replicas] AS ar
            ON ag.[group_id] = ar.[group_id]
        LEFT JOIN sys.[dm_hadr_availability_replica_states] AS ars
            ON ar.[replica_id] = ars.[replica_id]
    WHERE (ar.[read_only_routing_url] IS NULL
           OR ar.[read_only_routing_url] = N'')
        AND ars.[role] = 2 /* SECONDARY */

    UNION ALL

    /* Synchronous replica not in SYNCHRONIZED state */
    SELECT
        N'CRITICAL' AS [Severity]
       ,N'Sync Health' AS [Category]
       ,ag.[name]
       ,ar.[replica_server_name]
       ,NULL
       ,N'Synchronous commit replica is '
        + ars.[synchronization_health_desc]
        + N' — expected HEALTHY'
    FROM sys.[availability_groups] AS ag
        INNER JOIN sys.[availability_replicas] AS ar
            ON ag.[group_id] = ar.[group_id]
        INNER JOIN sys.[dm_hadr_availability_replica_states] AS ars
            ON ar.[replica_id] = ars.[replica_id]
    WHERE ar.[availability_mode] = 1 /* SYNCHRONOUS_COMMIT */
        AND ars.[synchronization_health] <> 2 /* NOT HEALTHY */
        AND ars.[role] = 2 /* SECONDARY */

    UNION ALL

    /* Disconnected replica */
    SELECT
        N'CRITICAL' AS [Severity]
       ,N'Connectivity' AS [Category]
       ,ag.[name]
       ,ar.[replica_server_name]
       ,NULL
       ,N'Replica is DISCONNECTED'
    FROM sys.[availability_groups] AS ag
        INNER JOIN sys.[availability_replicas] AS ar
            ON ag.[group_id] = ar.[group_id]
        INNER JOIN sys.[dm_hadr_availability_replica_states] AS ars
            ON ar.[replica_id] = ars.[replica_id]
    WHERE ars.[connected_state] = 0 /* DISCONNECTED */

    UNION ALL

    /* Large redo queue */
    SELECT
        N'WARNING' AS [Severity]
       ,N'Redo Lag' AS [Category]
       ,ag.[name]
       ,ar.[replica_server_name]
       ,adc.[database_name]
       ,N'Redo queue is '
        + CONVERT(nvarchar(20), drs.[redo_queue_size] / 1024)
        + N' MB — secondary is falling behind'
    FROM sys.[availability_groups] AS ag
        INNER JOIN sys.[availability_replicas] AS ar
            ON ag.[group_id] = ar.[group_id]
        INNER JOIN sys.[dm_hadr_database_replica_states] AS drs
            ON ar.[replica_id] = drs.[replica_id]
        INNER JOIN sys.[availability_databases_cluster] AS adc
            ON drs.[group_database_id] = adc.[group_database_id]
    WHERE drs.[redo_queue_size] > @RedoQueueThresholdKB

    UNION ALL

    /* Large log send queue */
    SELECT
        N'WARNING' AS [Severity]
       ,N'Send Lag' AS [Category]
       ,ag.[name]
       ,ar.[replica_server_name]
       ,adc.[database_name]
       ,N'Log send queue is '
        + CONVERT(nvarchar(20), drs.[log_send_queue_size] / 1024)
        + N' MB — check network throughput'
    FROM sys.[availability_groups] AS ag
        INNER JOIN sys.[availability_replicas] AS ar
            ON ag.[group_id] = ar.[group_id]
        INNER JOIN sys.[dm_hadr_database_replica_states] AS drs
            ON ar.[replica_id] = drs.[replica_id]
        INNER JOIN sys.[availability_databases_cluster] AS adc
            ON drs.[group_database_id] = adc.[group_database_id]
    WHERE drs.[log_send_queue_size] > @SendQueueThresholdKB

    UNION ALL

    /* Suspended data movement */
    SELECT
        N'CRITICAL' AS [Severity]
       ,N'Data Movement' AS [Category]
       ,ag.[name]
       ,ar.[replica_server_name]
       ,adc.[database_name]
       ,N'Data movement suspended — '
        + ISNULL(drs.[suspend_reason_desc], N'reason unknown')
    FROM sys.[availability_groups] AS ag
        INNER JOIN sys.[availability_replicas] AS ar
            ON ag.[group_id] = ar.[group_id]
        INNER JOIN sys.[dm_hadr_database_replica_states] AS drs
            ON ar.[replica_id] = drs.[replica_id]
        INNER JOIN sys.[availability_databases_cluster] AS adc
            ON drs.[group_database_id] = adc.[group_database_id]
    WHERE drs.[is_suspended] = 1
) AS findings
ORDER BY
    CASE [Severity]
        WHEN N'CRITICAL' THEN 1
        WHEN N'WARNING' THEN 2
        WHEN N'INFO' THEN 3
    END
   ,[AGName]
   ,[ReplicaServer];
```

## What I Validated

A few things the agent got right that I might have missed if I'd written this from scratch:

- **The `suspend_reason_desc` column.** When data movement is suspended, this tells you *why* — administrator action, partner shutdown, redo error, etc. The agent included it automatically, which saves a troubleshooting step.
- **Commit lag estimation.** The `last_commit_time` difference gives you a rough lag estimate, though it's not a precise SLA metric. The agent added a helpful comment noting that this is an approximation, not a guaranteed latency measurement.
- **Left joins on listener tables.** Not every AG has a listener (especially in dev environments or distributed AG configurations), so inner joins would silently hide those AGs.

One thing I added post-generation: a note that this query **must run on the primary replica** to see full topology. On a secondary, `sys.dm_hadr_availability_replica_states` only shows the local replica's state. The agent didn't mention this, and it's a common gotcha.

## Beyond the Query: Topology Documentation

One powerful follow-up is asking the agent to generate a human-readable topology document from the query output. I pasted a sample result set into the conversation and prompted:

```text
Given this AG topology data, generate a markdown document that describes
the AG configuration in plain language. Include a table per AG showing
replicas, their roles, sync modes, and any warnings. Format it so I can
paste it into our internal wiki.
```

The agent produced a clean document with tables and callouts for each misconfiguration. That's the kind of documentation that usually doesn't get written because nobody has time — but with the query output and one prompt, you've got it.

## Try This Yourself

Run the topology query on any instance with Availability Groups. Even if you think your AG configuration is clean, the misconfiguration detection section often surfaces surprises: a secondary that's been quietly accumulating redo queue, a read-only routing URL that points to a decommissioned server, or a replica someone switched to synchronous commit during a failover test and never switched back.

For deeper AG monitoring — health tracking over time rather than point-in-time snapshots — see the approach discussed in [Post 5](/ai-for-dbas/health-checks-inventory/). The point-in-time diagnostic here is a complement, not a replacement, for continuous monitoring.

---

*Part of the [ALTER DBA ADD AGENT](/ai-for-dbas/alter-dba-add-agent/) series.*
