# Snapshots

Snapshots load existing Postgres table data into Redis before pg2redis starts streaming new WAL changes.

Snapshots are useful when:

- Redis starts empty and must be initialized from Postgres.
- A new table mapping is added for a table that already contains data.
- WAL history is no longer available for the desired starting point.

## Snapshot Modes

### `onetime`

Run the snapshot once when no completed snapshot LSN exists in Redis.

This is the default mode when a snapshot config is present and `mode` is empty.

### `onetime_only`

Run the snapshot and exit without starting logical replication streaming.

### `never`

Do not run a snapshot. pg2redis starts streaming from the saved Redis LSN if present, otherwise from the replication slot position.

## Configuration

```yaml
snapshot:
  mode: onetime
  batchSize: 1000
  parallelWorkers: 4
  abortOnError: true
  queryTimeout: 30s
  maxWriteQueueSize: 10000
  config:
    - name: public.orders
      type: full
    - name: public.customers
      type: query
      query: "updated_at >= now() - interval '30 days'"
```

### `mode`, `PG2REDIS_SNAPSHOT_MODE`

Snapshot mode. Valid values are `onetime`, `onetime_only`, and `never`.

### `batchSize`, `PG2REDIS_SNAPSHOT_BATCHSIZE`

Target number of rows per snapshot chunk. Default is `1000`.

### `parallelWorkers`, `PG2REDIS_SNAPSHOT_PARALLELWORKERS`

Number of snapshot workers. Default is `1`.

Snapshot workers read Postgres ranges in parallel and submit Redis writes through a dedicated snapshot listener.

### `abortOnError`, `PG2REDIS_SNAPSHOT_ABORTONERROR`

When true, pg2redis exits if the snapshot finishes with errors.

When false, pg2redis records failed tables in `pgwalk.snapshot_state` and continues to streaming if possible.

### `queryTimeout`, `PG2REDIS_SNAPSHOT_QUERYTIMEOUT`

Timeout for snapshot planning queries, such as reading CTID bounds for filtered snapshots.

### `maxWriteQueueSize`, `PG2REDIS_SNAPSHOT_MAXWRITEQUEUESIZE`

Optional back-pressure limit used while dispatching snapshot ranges. When the snapshot listener write queue is above this value, pg2redis waits before adding more snapshot work.

### `config`

List of tables to snapshot.

Each table entry has:

- `name`: schema-qualified table name.
- `type`: `full` or `query`.
- `query`: SQL `WHERE` clause for `query` snapshots.

## Full Table Snapshot

```yaml
snapshot:
  mode: onetime
  config:
    - name: public.products
      type: full
```

This snapshots the full table.

pg2redis builds a query like:

```sql
SELECT <publication columns> FROM public.products
```

If the publication exposes all physical columns for the table, pg2redis can use `SELECT *`.

## Query Snapshot

```yaml
snapshot:
  mode: onetime
  config:
    - name: public.orders
      type: query
      query: "created_at >= '2026-01-01' and status <> 'cancelled'"
```

The `query` value is a SQL `WHERE` clause. Do not include the `WHERE` keyword.

pg2redis builds a query like:

```sql
SELECT <publication columns>
FROM public.orders
WHERE ( created_at >= '2026-01-01' and status <> 'cancelled' )
```

## How Snapshots Work

1. pg2redis validates the Postgres publication, replication slot, table mappings, and Redis connection.
2. If `postgres.repl.owner` is `app`, pg2redis creates or updates the publication and creates the replication slot if needed.
3. pg2redis prepares a consistent Postgres snapshot:
   - If a new replication slot was created by pg2redis, it uses the exported slot snapshot.
   - Otherwise it opens a repeatable-read, read-only transaction and calls `pg_export_snapshot()`.
4. pg2redis records the snapshot WAL LSN.
5. pg2redis creates or updates rows in `pgwalk.snapshot_state`.
6. Snapshot workers read table ranges in parallel.
7. Rows are sent through the same Redis command mapping as inserts.
8. pg2redis waits until all snapshot Redis writes are acknowledged.
9. The completed snapshot LSN is saved in Redis as `pgwalk:snapshot_lsn:<app_name>`.
10. If the mode is not `onetime_only`, pg2redis starts logical replication from the snapshot LSN or from a later saved stream LSN.

## Snapshot State Table

pg2redis stores snapshot progress in Postgres:

```sql
CREATE SCHEMA IF NOT EXISTS pgwalk;

CREATE TABLE IF NOT EXISTS pgwalk.snapshot_state (
    app_name TEXT NOT NULL,
    table_name TEXT NOT NULL,
    snapshot_lsn TEXT NOT NULL,
    snapshot_type TEXT NOT NULL,
    snapshot_query TEXT,
    resume_key_column TEXT,
    resume_key_value TEXT,
    total_rows BIGINT,
    processed_rows BIGINT,
    status TEXT NOT NULL,
    started_at TIMESTAMP,
    updated_at TIMESTAMP,
    snapshot_start TIMESTAMP,
    snapshot_end TIMESTAMP,
    error_message TEXT,
    CONSTRAINT pk_pgwal_snapshot_state PRIMARY KEY (app_name, table_name)
);
```

Valid statuses are:

- `pending`
- `in_progress`
- `completed`
- `failed`

The application creates this table automatically, but the Postgres user must have permission to create schema/table objects or the table must be created ahead of time.

## Monitoring Progress

Use this query to inspect current progress:

```sql
SELECT
  app_name,
  table_name,
  status,
  processed_rows,
  total_rows,
  snapshot_lsn,
  snapshot_start,
  snapshot_end,
  updated_at,
  error_message
FROM pgwalk.snapshot_state
ORDER BY table_name;
```

pg2redis also logs snapshot statistics at `statsInterval`, including pending, in-progress, completed, and failed table counts.

## Restart and Retry Behavior

For `onetime` mode, pg2redis first checks Redis key `pgwalk:snapshot_lsn:<app_name>`.

- If the key exists and contains a valid LSN, pg2redis skips the snapshot.
- If the key is missing, pg2redis checks `pgwalk.snapshot_state`.

For each configured table:

- `completed` tables are skipped.
- `pending`, `in_progress`, and `failed` tables are reset to `pending` and snapshotted again.

This means a partially completed snapshot can be resumed at table granularity. Individual table ranges are not resumed from the exact last row.

## Snapshot Planning

pg2redis uses Postgres table estimates and CTID ranges to split large tables into chunks.

For full snapshots, it scans CTID ranges across the table.

For query snapshots:

- Small filtered snapshots can run as one query.
- Sparse filtered snapshots can first read CTID bounds for matching rows.
- Larger filtered snapshots use CTID ranges with the configured `WHERE` clause.

The `batchSize` value is a target row count. Actual rows per batch can vary because Postgres CTID ranges are block-based.

## Redis Commands During Snapshot

Snapshot rows are treated as inserts.

For a table such as:

```yaml
tables:
  - public.orders:
      insert:
        commands:
          - ["HSET", "orders:{id}", "{pairs:*}"]
      update:
        commands:
          - ["HSET", "orders:{id}", "{pairs:*}"]
```

the snapshot uses the `insert.commands` section.

If `insert.commands` is not configured, pg2redis uses the table-level `commands`.

## Operational Notes

- Snapshot tables must be present in the Postgres publication.
- Columns referenced by Redis command templates and conditions must be present in publication metadata.
- Use `onetime_only` to pre-load Redis without leaving a streaming process running.
- Large snapshots hold a consistent Postgres snapshot open until the snapshot finishes. Monitor WAL retention and long-running transactions.
- For filtered snapshots, the `query` string is used directly as SQL. Keep it deterministic and test it before running in production.
- If Redis writes fail, inspect `error_message` in `pgwalk.snapshot_state`.
- To force `onetime` mode to enter the snapshot phase again, clear the Redis snapshot LSN key.
- To force any mode to reload an already completed table, update or clear that table's row in `pgwalk.snapshot_state`.
