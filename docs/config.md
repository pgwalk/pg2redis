# Configuration Guide

This document describes how to configure pg2redis using YAML configuration files or environment variables.

## Configuration Methods

The application can be configured using either:

1. YAML configuration file
2. Environment variables

Environment variables take precedence over YAML configuration values.

## YAML Configuration

The application loads configuration files in this order:

1. `/etc/pg2redis/config.yaml`
2. `./config.yaml`
3. File defined by `PG2REDIS_CONFIG_PATH`

If `PG2REDIS_CONFIG_PATH` is set and the file does not exist, the application exits.

If `ENV_FILE` is set, pg2redis loads that file before reading the application config.

## Environment Variables

Environment variables use the `PG2REDIS_` prefix. Names follow the YAML structure, with nested fields separated by underscores and written in uppercase.

Examples:

```bash
PG2REDIS_POSTGRES_CONN_HOST=localhost
PG2REDIS_POSTGRES_REPL_SLOT=pg2redis_slot
PG2REDIS_REDIS_CONN_HOST=localhost
PG2REDIS_FLUSHINTERVAL=500ms
```

## Main Configuration

### `flushInterval`, `PG2REDIS_FLUSHINTERVAL`

Interval for flushing pending Redis writes. Default is `500ms`. If the value has no unit, milliseconds are assumed.

### `flushBufferSize`, `PG2REDIS_FLUSHBUFFERSIZE`

Number of row changes buffered before a Redis batch is flushed. Default is `100`.

### `flushQueueDepth`, `PG2REDIS_FLUSHQUEUEDEPTH`

Depth of the Redis flush queue. Default and maximum is `32`.

### `flushWorkers`, `PG2REDIS_FLUSHWORKERS`

Number of parallel Redis flush workers. Default is based on max available CPUs and capped at `32`.

### Redis flush pipeline

`flushBufferSize`, `flushQueueDepth`, and `flushWorkers` tune different stages of the same Redis write pipeline.

1. pg2redis receives row changes from the replication listener.
2. Each row change is queued in the Redis writer input queue. `flushQueueDepth` controls the size of this input queue. If the queue is full, pg2redis applies back-pressure and waits before accepting more row changes into the Redis writer.
3. A batching loop drains the input queue and builds an in-memory Redis batch.
4. The batch is submitted when either:
   - the batch reaches `flushBufferSize` row changes, or
   - `flushInterval` elapses.
5. Submitted batches are executed as Redis pipelines. `flushWorkers` controls how many Redis pipelines can be executed in parallel.

In practical terms:

- Increase `flushBufferSize` to reduce Redis round trips and improve throughput. This can increase latency because rows may wait longer for a batch to fill.
- Decrease `flushBufferSize` or `flushInterval` to reduce latency. This usually increases Redis round trips.
- Increase `flushQueueDepth` to absorb short bursts before back-pressure starts. This does not change the size of a Redis batch.
- Increase `flushWorkers` when Redis and the network can handle multiple concurrent pipelines.
- Use `flushWorkers: 1` when strict Redis write ordering across batches is required for the same keys.

For example, with:

```yaml
flushBufferSize: 100
flushQueueDepth: 32
flushWorkers: 4
flushInterval: 500ms
```

pg2redis can hold up to 32 row changes in the Redis writer input queue, builds batches of up to 100 row changes, flushes partial batches after 500 ms, and executes up to 4 Redis pipelines at the same time.

### `maxWriteQueueSize`, `PG2REDIS_MAXWRITEQUEUESIZE`

Maximum number of pending replication write requests. When the queue reaches this limit, pg2redis finishes the current transaction and then waits before generating more downstream writes.

### `writeTimeout`, `PG2REDIS_WRITETIMEOUT`

Timeout for Redis read/write operations. Default is `10s`. If the value has no unit, seconds are assumed.

### `shutdownTimeout`, `PG2REDIS_SHUTDOWNTIMEOUT`

Timeout for graceful shutdown. If the value has no unit, milliseconds are assumed.

### `statsInterval`, `PG2REDIS_STATSINTERVAL`

Interval for logging application statistics. Default is `1m`. If the value has no unit, seconds are assumed.

### `retryPolicy`

Retry policy for failed downstream operations.

The retry policy controls what pg2redis does when a Redis write fails after a Postgres change has already been read.

Retryable failures are scheduled again after a backoff delay. Non-retryable Redis errors, such as `WRONGTYPE`, unknown command, and Redis syntax errors, are skipped and logged because retrying the same command is not expected to fix them.

If `retryPolicy` is not configured, pg2redis does not schedule delayed retries.

#### `maxRetries`, `PG2REDIS_RETRYPOLICY_MAXRETRIES`

Maximum number of retry attempts. Must be greater than or equal to `1`.

#### `maxConnectionRetries`, `PG2REDIS_RETRYPOLICY_MAXCONNECTIONRETRIES`

Maximum number of connection retry attempts. A value of `0` means connection retries continue without a fixed limit.

#### `initialBackoff`, `PG2REDIS_RETRYPOLICY_INITIALBACKOFF`

Initial wait time before retrying. Must be greater than `0`. If the value has no unit, seconds are assumed.

#### `multiplier`, `PG2REDIS_RETRYPOLICY_MULTIPLIER`

Backoff multiplier between retries. Must be greater than or equal to `1`.

#### `jitter`, `PG2REDIS_RETRYPOLICY_JITTER`

Random variation applied to retry backoff. Must be in the `0.0` to `1.0` range.

#### `maxBackoff`, `PG2REDIS_RETRYPOLICY_MAXBACKOFF`

Maximum retry backoff duration. Must be greater than `0`.

### Retry policy examples

#### Balanced production retries

```yaml
retryPolicy:
  maxRetries: 5
  maxConnectionRetries: 20
  initialBackoff: 1s
  multiplier: 2
  jitter: 0.2
  maxBackoff: 30s
```

This starts with short retries and then backs off up to 30 seconds. With `jitter: 0.2`, each retry delay can vary by up to 20 percent, which helps avoid many WAL entries retrying at the same time.

Example backoff sequence before jitter:

```text
2s, 4s, 8s, 16s, 30s
```

#### Keep retrying Redis outages

```yaml
retryPolicy:
  maxRetries: 10
  maxConnectionRetries: 0
  initialBackoff: 5s
  multiplier: 1.5
  jitter: 0.3
  maxBackoff: 60s
```

Use this when Redis outages should not stop the process automatically. `maxConnectionRetries: 0` means connection-style downstream failures keep retrying without a fixed limit. Individual entries can still be skipped when they hit `maxRetries` for retryable non-connection failures.

#### Fail fast when Redis is unavailable

```yaml
retryPolicy:
  maxRetries: 3
  maxConnectionRetries: 3
  initialBackoff: 1s
  multiplier: 2
  jitter: 0
  maxBackoff: 10s
```

Use this when an orchestrator should restart or alert quickly if Redis is unavailable. After repeated downstream-down failures, pg2redis exits instead of retrying forever.

#### Low-latency retries

```yaml
retryPolicy:
  maxRetries: 3
  maxConnectionRetries: 10
  initialBackoff: 100ms
  multiplier: 2
  jitter: 0.1
  maxBackoff: 2s
```

Use this for short transient failures where retry latency matters more than reducing load on Redis.

#### Linear backoff

```yaml
retryPolicy:
  maxRetries: 5
  maxConnectionRetries: 0
  initialBackoff: 2s
  multiplier: 1
  jitter: 0
  maxBackoff: 20s
```

When `multiplier` is `1`, pg2redis uses a linear backoff based on the retry count. In this example, the delay grows roughly as:

```text
2s, 4s, 6s, 8s, 10s
```

### Choosing retry values

- Use a larger `initialBackoff` and `maxBackoff` when Redis failures usually mean the service is overloaded.
- Use a smaller `initialBackoff` when most failures are brief network hiccups.
- Use `jitter` greater than `0` to avoid thundering herds of retries when Redis is overloaded.
- Use `maxConnectionRetries: 0` only when you want pg2redis to keep waiting for Redis indefinitely.
- Keep `maxRetries` finite so bad data or command-level failures do not block WAL progress forever.

## Postgres Configuration, `postgres`

### Connection settings, `postgres.conn`

#### `host`, `PG2REDIS_POSTGRES_CONN_HOST`

Postgres host. Can be a comma-separated list of hosts. pg2redis connects to the active primary node.

#### `port`, `PG2REDIS_POSTGRES_CONN_PORT`

Postgres port. Can be one port for all hosts, or a comma-separated list with the same number of entries as `host`.

#### `database`, `PG2REDIS_POSTGRES_CONN_DATABASE`

Database name.

#### `user`, `PG2REDIS_POSTGRES_CONN_USER`

Database user.

#### `password`, `PG2REDIS_POSTGRES_CONN_PASSWORD`

Database password. A password is required unless TLS settings are configured.

### TLS settings, `postgres.conn.tls`

#### `cert`, `PG2REDIS_POSTGRES_CONN_TLS_CERT`

Client certificate path.

#### `key`, `PG2REDIS_POSTGRES_CONN_TLS_KEY`

Client key path.

#### `rootCert`, `PG2REDIS_POSTGRES_CONN_TLS_ROOTCERT`

Root CA certificate path.

If one TLS field is set, all three must be set.

### Replication settings, `postgres.repl`

#### `pub`, `PG2REDIS_POSTGRES_REPL_PUB`

Publication name.

#### `slot`, `PG2REDIS_POSTGRES_REPL_SLOT`

Logical replication slot name. The slot must use the `pgoutput` plugin.

#### `owner`, `PG2REDIS_POSTGRES_REPL_OWNER`

Controls who manages the publication and slot.

- `app`: pg2redis creates or updates the publication and creates the slot when missing.
- `user`: you create and manage the publication and slot manually.

When `owner` is `app`, the database user needs enough privileges to create or alter the publication and create the replication slot.

#### `failover`, `PG2REDIS_POSTGRES_REPL_FAILOVER`

Controls failover-enabled logical slot creation when pg2redis creates a slot. Failover slots require Postgres 17 or newer. If omitted, pg2redis enables failover slot creation automatically on Postgres 17 or newer.

### `numericMode`, `PG2REDIS_POSTGRES_NUMERICMODE`

Controls how Postgres floating point and arbitrary precision numeric values are decoded before they are written to Redis.

This setting applies to Postgres `real`, `double precision`, and `numeric`/`decimal` columns. Integer columns such as `smallint`, `integer`, and `bigint` are decoded as integers regardless of `numericMode`.

- `float`: values are decoded as numeric values. This is the default.
- `string`: values are decoded from the Postgres text representation and preserved as strings.

Default is `float`.

### `standByTimeout`, `PG2REDIS_POSTGRES_STANDBYTIMEOUT`

Standby status update interval for logical replication. If not set, pg2redis uses Postgres `wal_sender_timeout`. If configured above `wal_sender_timeout`, pg2redis lowers it to `wal_sender_timeout`.

### `receiveTimeout`, `PG2REDIS_POSTGRES_RECEIVETIMEOUT`

Timeout for receiving messages from the replication stream. Default is `60s`.

## Redis Configuration, `redis`

### Connection settings, `redis.conn`

#### `host`, `PG2REDIS_REDIS_CONN_HOST`

Redis host.

#### `port`, `PG2REDIS_REDIS_CONN_PORT`

Redis port.

#### `database`, `PG2REDIS_REDIS_CONN_DATABASE`

Redis database index.

#### `user`, `PG2REDIS_REDIS_CONN_USER`

Redis username.

#### `password`, `PG2REDIS_REDIS_CONN_PASSWORD`

Redis password.

### TLS settings, `redis.conn.tls`

#### `cert`, `PG2REDIS_REDIS_CONN_TLS_CERT`

Client certificate path.

#### `key`, `PG2REDIS_REDIS_CONN_TLS_KEY`

Client key path.

#### `rootCert`, `PG2REDIS_REDIS_CONN_TLS_ROOTCERT`

Root CA certificate path.

If one TLS field is set, all three must be set.

### `commitTimeColumn`, `PG2REDIS_REDIS_COMMITTIMECOLUMN`

Adds the Postgres transaction commit time to generated Redis payloads.

This option requires Postgres `track_commit_timestamp` to be enabled. You can enable it with:

```sql
ALTER SYSTEM SET track_commit_timestamp = ON;
```

Postgres must be restarted after changing `track_commit_timestamp`.

For Redis command templates, the commit time is automatically included by `{pairs:*}`, `{json:*}`, and `{columns:*}`. It is not automatically added to explicit subset templates such as `{pairs:id,status}` or `{json:id,status}`.

## Table Configuration, `tables`

Each table entry maps one Postgres table to one or more Redis commands.

```yaml
tables:
  - public.orders:
      commands:
        - ["HSET", "orders:{id}", "{pairs:*}"]
```

If the table name has no schema, `public` is assumed.

See [Redis commands](redis-commands.md) for command templates, operation-specific commands, conditions, and examples.

### Table environment variables

Tables are configured with indexed environment variables:

```bash
PG2REDIS_T_0_NAME=public.orders
PG2REDIS_T_0_COMMANDS_0=HSET/orders:{id}/{pairs:*}

PG2REDIS_T_1_NAME=public.customers
PG2REDIS_T_1_INSERT_COMMANDS_0=HSET/customers:{id}/{pairs:*}
PG2REDIS_T_1_DELETE_COMMANDS_0=DEL/customers:{id}
```

Command arguments in simple environment variables are separated with `/`.

Conditional commands can also be represented in environment variables:

```bash
PG2REDIS_T_0_COMMANDS_0_COMMAND_0=HSET
PG2REDIS_T_0_COMMANDS_0_COMMAND_1=orders:{id}
PG2REDIS_T_0_COMMANDS_0_COMMAND_2={pairs:*}
PG2REDIS_T_0_COMMANDS_0_CONDITION_COLUMN=status
PG2REDIS_T_0_COMMANDS_0_CONDITION_OP="="
PG2REDIS_T_0_COMMANDS_0_CONDITION_VALUE=active
```

YAML is recommended for complex table mappings because multi-command groups and conditions are easier to read.

## Snapshot Configuration, `snapshot`

Snapshots are configured under `snapshot`.

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

Snapshot environment variables:

```bash
PG2REDIS_SNAPSHOT_MODE=onetime
PG2REDIS_SNAPSHOT_BATCHSIZE=1000
PG2REDIS_SNAPSHOT_PARALLELWORKERS=4
PG2REDIS_SNAPSHOT_ABORTONERROR=true
PG2REDIS_SNAPSHOT_QUERYTIMEOUT=30s
PG2REDIS_SNAPSHOT_MAXWRITEQUEUESIZE=10000
```

See [Snapshots](snapshot.md) for details.

## License

Set the license key directly:

```bash
PGWALK_LIC_PG2REDIS=<license key>
```

Or use a license file:

```bash
PGWALK_LICFILE_PG2REDIS=/path/to/key.pg2redislic
```

The application will not start without a valid license.

## Example Configuration

```yaml
postgres:
  conn:
    host: localhost
    port: 5432
    database: tickets
    user: postgres
    password: password
  repl:
    pub: tickets_pub
    slot: tickets_slot
    owner: app
  numericMode: string
  standByTimeout: 30s
  receiveTimeout: 60s

redis:
  conn:
    host: localhost
    port: 6379
    database: 0
    password: ""
  commitTimeColumn: lastupdated

writeTimeout: 25s
flushInterval: 500ms
flushBufferSize: 150
flushQueueDepth: 32
flushWorkers: 4
statsInterval: 1m
maxWriteQueueSize: 10000
shutdownTimeout: 30s
debug: false

retryPolicy:
  maxRetries: 5
  maxConnectionRetries: 0
  initialBackoff: 1s
  multiplier: 2
  jitter: 0.2
  maxBackoff: 30s

tables:
  - public.venues:
      insert:
        commands:
          - ["HSET", "venue:{public_id}", "{pairs:*}"]
      update:
        commands:
          - ["HSET", "venue:{public_id}", "{pairs:*}"]
      delete:
        commands:
          - ["DEL", "venue:{public_id}"]

  - public.event_seats:
      insert:
        commands:
          - ["HSET", "event:{event_id}:section:{section}:seats", "{seat_id}", "{json:*}"]
          - ["HINCRBY", "event:{event_id}:summary", "section:{section}:total", "1"]
          - command: ["HINCRBY", "event:{event_id}:summary", "section:{section}:available", "1"]
            condition:
              column: status
              op: "="
              value: A
      update:
        commands:
          - ["HSET", "event:{event_id}:section:{section}:seats", "{seat_id}", "{json:*}"]
          - command: ["HINCRBY", "event:{event_id}:summary", "section:{section}:available", "-1"]
            condition:
              column: status
              op: "<>"
              value: A

snapshot:
  mode: onetime
  batchSize: 1000
  parallelWorkers: 4
  abortOnError: true
  queryTimeout: 30s
  maxWriteQueueSize: 10000
  config:
    - name: public.venues
      type: full
    - name: public.event_seats
      type: query
      query: "created_at >= '2026-01-01'"
```

## Time Units

Duration settings accept Go duration units: `ns`, `us`, `ms`, `s`, `m`, and `h`.

Some settings also accept plain numbers:

- `flushInterval`: plain numbers are milliseconds.
- `shutdownTimeout`: plain numbers are milliseconds.
- `writeTimeout`, `statsInterval`, `standByTimeout`, `receiveTimeout`, `retryPolicy.initialBackoff`, `retryPolicy.maxBackoff`: plain numbers are seconds.

## Numeric Modes

`numericMode` affects the values that pg2redis stores in the in-memory row tuple before Redis command templates are expanded.

It is most visible when you use JSON templates such as `{json:*}` or `{json:price,score}`. Redis hash/set command arguments are strings by the time they are sent to Redis, but `numericMode` still controls whether pg2redis formats a decoded numeric value or preserves the text value Postgres sends.

### `float`

`float` is the default mode.

In this mode:

- `real` and `double precision` are decoded as floating point values.
- `numeric` and `decimal` are decoded as Postgres numeric values and are emitted as JSON numbers.
- `NaN` is emitted as `null` in JSON.
- `Infinity` for `real`/`double precision` is emitted as the largest representable float value in JSON.
- `-Infinity` for `real`/`double precision` is emitted as the smallest positive non-zero float value in JSON.

Example table:

```sql
CREATE TABLE product_prices (
    id bigint PRIMARY KEY,
    price numeric(20,10),
    rating real,
    distance double precision,
    quantity integer
);
```

Configuration:

```yaml
postgres:
  numericMode: float

tables:
  - public.product_prices:
      commands:
        - ["SET", "product_prices:{id}", "{json:*}"]
```

For this row:

```sql
INSERT INTO product_prices(id, price, rating, distance, quantity)
VALUES (1, 1.23, 4.5, 100.125, 7);
```

the JSON stored by Redis `SET product_prices:1 ...` is:

```json
{
  "id": 1,
  "price": 1.2300000000,
  "rating": 4.5,
  "distance": 100.125,
  "quantity": 7
}
```

This mode is convenient when downstream readers expect JSON numbers, but it can lose floating point precision for `real` and `double precision`.

### `string`

In this mode:

- `real`, `double precision`, `numeric`, and `decimal` are decoded as strings.
- Integer columns remain JSON numbers.
- `NaN`, `Infinity`, and `-Infinity` are preserved as strings when Postgres sends them for the affected types.

Configuration:

```yaml
postgres:
  numericMode: string

tables:
  - public.product_prices:
      commands:
        - ["SET", "product_prices:{id}", "{json:*}"]
```

For the same row:

```sql
INSERT INTO product_prices(id, price, rating, distance, quantity)
VALUES (1, 1.23, 4.5, 100.125, 7);
```

the JSON stored by Redis `SET product_prices:1 ...` is:

```json
{
  "id": 1,
  "price": "1.2300000000",
  "rating": "4.5",
  "distance": "100.125",
  "quantity": 7
}
```

This mode is safest when precision matters, especially for `numeric`/`decimal` values used for money, balances, rates, scores, or identifiers that must not be rounded.

### Special values example

For this row:

```sql
INSERT INTO product_prices(id, price, rating, distance, quantity)
VALUES (2, 'NaN', 'Infinity', '-Infinity', 1);
```

`numericMode: float` produces JSON like:

```json
{
  "id": 2,
  "price": null,
  "rating": 3.4028234663852886e+38,
  "distance": 5e-324,
  "quantity": 1
}
```

`numericMode: string` produces JSON like:

```json
{
  "id": 2,
  "price": "NaN",
  "rating": "Infinity",
  "distance": "-Infinity",
  "quantity": 1
}
```

### Hash example

With a hash mapping:

```yaml
postgres:
  numericMode: string

tables:
  - public.product_prices:
      commands:
        - ["HSET", "product_prices:{id}", "{pairs:*}"]
```

Redis receives field/value pairs like:

```text
HSET product_prices:1 id 1 price 1.2300000000 rating 4.5 distance 100.125 quantity 7
```

Redis stores hash values as strings, so `numericMode` is less visible for hashes than for JSON payloads. Use `string` mode when you want the text representation from Postgres preserved before pg2redis formats Redis command arguments.

### Condition and diff behavior

Conditional commands use the decoded values. Numeric comparison operators such as `<`, `>`, `<=`, and `>=` can compare values numerically when both sides can be parsed as numbers.

```yaml
tables:
  - public.product_prices:
      commands:
        - command: ["SADD", "expensive_products", "{id}"]
          condition:
            column: price
            op: ">"
            value: "100.00"
```

For equality conditions in `string` mode, use the exact text representation Postgres sends. For example, a `numeric(20,10)` value may appear as `"1.2300000000"`, not `"1.23"`.

`{diff:column}` converts values to floats internally, so it is useful for counters and approximate score changes, not for exact decimal arithmetic.
