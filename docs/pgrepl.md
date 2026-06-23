# Replication Management

pg2redis relies on Postgres logical replication. It uses publications to decide which tables and columns are available, and a logical replication slot to read changes from WAL.

pg2redis uses the built-in `pgoutput` plugin.

## Postgres Requirements

Minimum supported Postgres version is 12.

Configure Postgres for logical replication:

```conf
wal_level = logical
max_replication_slots = 10
max_wal_senders = 10
```

Restart Postgres after changing these settings.

If you use `redis.commitTimeColumn`, also enable:

```conf
track_commit_timestamp = on
```

Changing `track_commit_timestamp` requires a Postgres restart.

## Publications

A publication defines which table changes Postgres sends to pg2redis.

```sql
CREATE PUBLICATION pg2redis_pub FOR TABLE public.orders, public.customers;
```

For Postgres 15 and newer, publications can include a subset of columns:

```sql
CREATE PUBLICATION pg2redis_pub
FOR TABLE public.orders (id, status, total);
```

Columns referenced by Redis command templates and conditions must be available in the publication.

## Replication Slots

A logical replication slot keeps WAL available until pg2redis reads it.

```sql
SELECT pg_create_logical_replication_slot('pg2redis_slot', 'pgoutput');
```

The slot must be created with the `pgoutput` plugin. pg2redis refuses to start if the configured slot uses a different plugin.

Monitor replication slots. An unused or stalled slot can retain WAL and consume disk space.

## Application-Owned Replication Setup

When `postgres.repl.owner` is `app`, pg2redis manages the publication and slot on the detected primary during startup:

```yaml
postgres:
  repl:
    pub: pg2redis_pub
    slot: pg2redis_slot
    owner: app
```

In this mode pg2redis:

- Creates the publication if it does not exist.
- Alters the publication to match configured tables and columns.
- Creates the logical replication slot if it does not exist.
- Uses `publish_via_partition_root=true` for new publications on Postgres 13 and newer.

The Postgres user needs enough privileges to perform these operations.

## User-Owned Replication Setup

When `postgres.repl.owner` is `user`, you manage the publication and slot manually:

```yaml
postgres:
  repl:
    pub: pg2redis_pub
    slot: pg2redis_slot
    owner: user
```

In this mode pg2redis validates that the publication and slot exist and that the slot uses `pgoutput`.

## Failover Slots

Postgres 17 added failover support for logical replication slots.

When pg2redis creates a slot and `postgres.repl.failover` is omitted, it enables failover slot creation automatically on Postgres 17 or newer.

You can also set it explicitly:

```yaml
postgres:
  repl:
    slot: pg2redis_slot
    failover: true
```

Setting `failover: true` on Postgres versions older than 17 is invalid.

## Multiple Postgres Hosts

Multiple Postgres hosts are used for primary discovery and failover-aware startup. Instead of pointing pg2redis at only one server, you can provide all known Postgres nodes in the cluster. pg2redis connects to the configured nodes, detects which one is currently primary, and uses that primary for publication validation, snapshot queries, slot management, and logical replication.

`postgres.conn.host` can be a comma-separated list of hosts:

```yaml
postgres:
  conn:
    host: pg-primary,pg-replica
    port: 5432
```

You can also provide one port per host:

```yaml
postgres:
  conn:
    host: pg-primary,pg-replica
    port: 5432,5433
```

The number of ports must be either one or equal to the number of hosts.

### Why use multiple hosts

Use multiple hosts when Postgres may be promoted from one node to another, for example in a primary/replica setup managed by Patroni, repmgr, cloud failover, or manual promotion.

With multiple hosts configured:

- pg2redis does not rely on the first host being the primary.
- A replica can be unavailable while the primary is still reachable.
- After a promotion, pg2redis can reconnect and discover the newly promoted primary.
- When replicas are configured and supported, pg2redis can try to advance matching logical replication slots on replica nodes.

### How primary detection works

For each configured host/port pair, pg2redis opens a normal Postgres connection and runs:

```sql
select pg_is_in_recovery();
```

A node where `pg_is_in_recovery()` returns `false` is treated as the primary. A node where it returns `true` is treated as a replica.

pg2redis connects to all configured nodes concurrently during discovery. Connection errors for individual nodes are logged. Startup can continue as long as at least one reachable node is detected as primary. If no primary is found, startup fails with a primary-node connection error.

After the primary is found:

- Normal SQL work is run on the primary.
- Snapshot reads are run on the primary.
- Publication and slot migration are run on the detected primary during startup when `postgres.repl.owner` is `app`.
- The logical replication connection is opened to the primary.

### Host and port parsing

Whitespace around comma-separated hosts and ports is ignored.

One port applies to all hosts:

```yaml
postgres:
  conn:
    host: pg-a, pg-b, pg-c
    port: 5432
```

This expands to:

```text
pg-a:5432
pg-b:5432
pg-c:5432
```

If each host has a different port, provide the same number of ports as hosts:

```yaml
postgres:
  conn:
    host: pg-a, pg-b, pg-c
    port: 5432, 5433, 5434
```

This expands to:

```text
pg-a:5432
pg-b:5433
pg-c:5434
```

The configuration is invalid when there is more than one port and the host count does not match the port count.

### Replication slot state on replicas

When replicas are configured, pg2redis tries to keep replica slot state close to the primary slot state by periodically running this on replica nodes:

```sql
select pg_replication_slot_advance($1, $2);
```

This is best-effort. If the replica slot advance fails, pg2redis logs the error and continues processing from the primary.

Replica slot state sync is enabled only when all configured nodes are Postgres 16 or newer. If any configured node is older than Postgres 16, pg2redis disables replica slot sync and logs a warning.

The matching logical slot must already exist on the replica, or be maintained by native Postgres slot synchronization. pg2redis does not create replica slots in this path; it only tries to advance existing slots.

For Postgres 17 or newer, prefer native Postgres failover logical slot support when possible. See [Failover Slots](#failover-slots).

### Slot and publication creation during failover

`postgres.repl.owner: app` does not mean pg2redis prepares every configured host in advance.

When pg2redis starts, it detects the current primary. If `postgres.repl.owner` is `app`, it creates or updates the publication on that primary and creates the logical slot on that primary when the slot is missing.

During a replication reconnect, pg2redis can re-detect a newly promoted primary, but it does not run publication or slot migration again in that reconnect path. It opens a replication connection to the current primary and tries to start replication from the saved LSN.

This means a standby that is promoted should already have the required publication and a usable logical slot before pg2redis relies on it as the new primary. That can be handled outside pg2redis, or by native Postgres failover slot synchronization on Postgres 17 or newer. If pg2redis is fully restarted after promotion and `postgres.repl.owner` is `app`, startup migration may create a missing publication or slot on the new primary, but a newly created slot starts from its own creation point and cannot replay changes that occurred before that slot existed.

### Limitations

- Multi-host configuration is not a load-balancing feature. pg2redis streams from exactly one primary node.
- All configured hosts must point to the same Postgres cluster and database.
- All hosts share the same configured database, user, password, and TLS settings.
- pg2redis must be allowed through `pg_hba.conf` on every host for normal SQL connections. The current primary must also allow replication connections.
- `postgres.repl.owner: app` creates or updates replication objects only on the primary detected during startup. It does not proactively create publications or slots on standby hosts.
- If a promoted node does not already have a valid logical slot at the needed WAL position, pg2redis cannot recover changes that happened before a new slot is created.
- Replica slot state sync requires Postgres 16 or newer on all configured nodes and is best-effort.
- pg2redis does not create logical slots on replicas as part of replica slot state sync.
- pg2redis does not stream WAL from replicas; replicas are used for discovery and best-effort slot-state advancement.
- During failover, pg2redis can reconnect and re-detect the primary, but an in-flight replication connection can still terminate the process for some replication read errors. Run pg2redis under a supervisor or orchestrator that restarts it after failover.
- If two configured nodes both report as primary, for example during split-brain, pg2redis uses the first primary it finds in the configured order. Fix the Postgres cluster state before running pg2redis.

## Permissions

Create a user with replication and login permissions:

```sql
CREATE USER pg2redis_user WITH REPLICATION LOGIN PASSWORD 'secure_password';
```

Grant read access to tables that pg2redis snapshots or streams:

```sql
GRANT SELECT ON public.orders TO pg2redis_user;
GRANT SELECT ON public.customers TO pg2redis_user;
```

If pg2redis owns publication management, grant the privileges needed to create or alter publications.

If pg2redis owns snapshot state creation, allow it to create the `pgwalk` schema and `pgwalk.snapshot_state` table, or create them ahead of time.

The host running pg2redis must also be allowed to connect through `pg_hba.conf` for both normal database connections and replication connections.

## Application State

Streaming LSN state is stored in Redis key:

```text
pg2redis_lsn
```

Completed snapshot LSN state is stored in Redis key:

```text
pgwalk:snapshot_lsn:pg2redis
```

pg2redis also attempts to create `pgwalk.app_state` in Postgres:

```sql
CREATE SCHEMA IF NOT EXISTS pgwalk;

CREATE TABLE IF NOT EXISTS pgwalk.app_state
(
 app_name TEXT NOT NULL PRIMARY KEY,
 state TEXT NOT NULL,
 last_xid TEXT NOT NULL
);
```

Grant access if you create it manually:

```sql
GRANT USAGE ON SCHEMA pgwalk TO pg2redis_user;
GRANT SELECT, INSERT, UPDATE, DELETE ON pgwalk.app_state TO pg2redis_user;
```

## Replica Identity

Deletes and previous-value comparisons depend on what Postgres includes in logical replication messages.

For tables where delete commands or `{old:column}` placeholders need non-key columns, configure replica identity:

```sql
ALTER TABLE public.orders REPLICA IDENTITY FULL;
```

Use this carefully on high-write tables because it increases WAL volume.

## Limitations

- Logical replication can publish a change before the same change is visible on an asynchronous standby.
- Postgres versions before 15 do not support column-list publications.
- Generated column replication has Postgres version-specific behavior. Verify generated columns before using them in Redis templates.
- Replication slots retain WAL until consumed; monitor disk usage and slot lag.
