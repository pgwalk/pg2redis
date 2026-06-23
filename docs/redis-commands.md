# Redis Commands

pg2redis converts Postgres row changes into Redis commands. Each configured table can define commands that run for all operations, or separate commands for inserts, updates, and deletes.

## Basic Table Mapping

```yaml
tables:
  - public.orders:
      commands:
        - ["HSET", "orders:{id}", "{pairs:*}"]
```

With this configuration, inserts and updates write all row columns into a Redis hash named `orders:<id>`.

If no commands are configured for a table, pg2redis defaults to:

```yaml
["HSET", "<schema.table>:{%pk%}", "{pairs:*}"]
```

For predictable cleanup, define delete commands explicitly.

## Operation-Specific Commands

Use `insert`, `update`, and `delete` to configure different Redis behavior for each Postgres operation.

```yaml
tables:
  - public.products:
      insert:
        commands:
          - ["HSET", "products:{id}", "{pairs:*}"]
          - ["SADD", "products:all", "{id}"]
      update:
        commands:
          - ["HSET", "products:{id}", "{pairs:*}"]
      delete:
        commands:
          - ["DEL", "products:{id}"]
          - ["SREM", "products:all", "{id}"]
```

For snapshot rows, pg2redis uses insert commands. If `insert.commands` is not configured, it uses the top-level `commands`.

## Supported Redis Commands

The configuration validator accepts these Redis commands:

```text
SET, SETEX, MSET, INCR, DECR, INCRBY, DECRBY, APPEND, DEL, EXPIRE,
HSET, HINCRBY, HDEL,
SADD, SREM,
ZADD, ZINCRBY, ZREM,
XADD, XDEL,
PUBLISH
```

Command names are case-insensitive in configuration and are sent to Redis in uppercase.

## Command Templates

Every command is an array. The first item is the Redis command name, and the remaining items are command arguments.

Template placeholders are expanded for each row change.

### Column placeholders

```text
{id}
{status}
{customer_id}
```

A column placeholder is replaced with the current row value.

### Previous-value placeholders

```text
{old:status}
{old:score}
```

Previous-value placeholders read from the previous tuple when Postgres provides it. If the previous value is unavailable, the placeholder expands to an empty string in command templates.

### Special placeholders

```text
{%schema%}  Postgres schema name
{%table%}   Postgres table name without schema
{%pk%}      joined primary key value
{%xid%}     Postgres transaction ID
```

Example:

```yaml
["HSET", "{%schema%}.{%table%}:{%pk%}", "{pairs:*}"]
```

### Literal braces

Use doubled braces for literal braces:

```yaml
["SET", "debug:{id}", "{{literal}}"]
```

## Structured Template Arguments

### `{pairs:*}`

Expands to Redis field/value pairs for all configured columns.

```yaml
["HSET", "orders:{id}", "{pairs:*}"]
```

Example Redis command:

```text
HSET orders:42 id 42 status paid total 19.99
```

### `{pairs:col1,col2}`

Expands to field/value pairs for selected columns.

```yaml
["HSET", "orders:{id}", "{pairs:id,status,total}"]
```

### `{json:*}`

Expands to a JSON object containing all configured columns.

```yaml
["SET", "orders:{id}", "{json:*}"]
["PUBLISH", "orders", "{json:*}"]
```

### `{json:col1,col2}`

Expands to a JSON object containing selected columns.

```yaml
["SET", "orders:{id}", "{json:id,status,total}"]
```

### `{columns:*}`

Expands to a list of column names. This is commonly used with `HDEL`.

```yaml
delete:
  commands:
    - ["HDEL", "orders:{id}", "{columns:*}"]
```

### `{diff:column}`

Expands to the numeric difference between the current and previous value of a column.

```yaml
update:
  commands:
    - ["ZINCRBY", "product_scores", "{diff:score}", "{product_id}"]
```

This is useful for counters and sorted sets. If no previous value is available, pg2redis treats it as zero.

## Commit Time

When `redis.commitTimeColumn` is configured, pg2redis can include transaction commit time in generated payloads.

```yaml
redis:
  commitTimeColumn: lastupdated
```

This requires Postgres `track_commit_timestamp` to be enabled.

The commit time is automatically included by:

- `{pairs:*}`
- `{json:*}`
- `{columns:*}`

Explicit subset templates such as `{pairs:id,status}` and `{json:id,status}` do not automatically include the commit time field.

## Conditional Commands

A command can be wrapped in a condition. The command runs only when the condition matches the row.

```yaml
tables:
  - public.orders:
      commands:
        - command: ["HSET", "orders:active:{id}", "{pairs:*}"]
          condition:
            column: status
            op: "="
            value: active
        - command: ["HSET", "orders:cancelled:{id}", "{pairs:*}"]
          condition:
            column: status
            op: in
            values: [cancelled, refunded]
```

Supported operators:

```text
=
<>
<
>
<=
>=
in
not_in
is_null
is_not_null
is_distinct_from
```

### Condition Values

Operators `=`, `<>`, `<`, `>`, `<=`, `>=`, and `is_distinct_from` use `value`.

```yaml
condition:
  column: status
  op: is_distinct_from
  value: "{old:status}"
```

Operators `in` and `not_in` use `values`.

```yaml
condition:
  column: status
  op: not_in
  values: [cancelled, completed]
```

Operators `is_null` and `is_not_null` do not use `value` or `values`.

```yaml
condition:
  column: deleted_at
  op: is_null
```

Condition values can be literals or column macros such as `{old:status}`.

## Multi-Command Groups

A single conditional command can contain multiple Redis commands. If the condition matches, pg2redis runs the whole group.

```yaml
tables:
  - public.orders:
      commands:
        - command:
            - ["HSET", "orders:{id}", "{pairs:*}"]
            - ["PUBLISH", "order_updates", "{id}"]
          condition:
            column: status
            op: "="
            value: completed
```

When more than one Redis command is produced for a row change, pg2redis wraps the commands in Redis `MULTI` and `EXEC`.

## Common Patterns

### Hash Per Row

```yaml
tables:
  - public.customers:
      insert:
        commands:
          - ["HSET", "customers:{id}", "{pairs:*}"]
      update:
        commands:
          - ["HSET", "customers:{id}", "{pairs:*}"]
      delete:
        commands:
          - ["DEL", "customers:{id}"]
```

### JSON Value Per Row

```yaml
tables:
  - public.orders:
      commands:
        - ["SET", "orders:{id}", "{json:*}"]
      delete:
        commands:
          - ["DEL", "orders:{id}"]
```

### Redis Stream

```yaml
tables:
  - public.orders:
      commands:
        - ["XADD", "orders_stream", "*", "{pairs:*}"]
```

### Pub/Sub Notification

```yaml
tables:
  - public.orders:
      commands:
        - ["PUBLISH", "orders", "{json:*}"]
```

### Sorted Set Counter

```yaml
tables:
  - public.product_events:
      insert:
        commands:
          - ["ZINCRBY", "product_popularity", "1", "{product_id}"]
      delete:
        commands:
          - ["ZINCRBY", "product_popularity", "-1", "{product_id}"]
```

## Usage Notes

- pg2redis validates command placeholders against publication metadata on startup.
- Primary key columns must be available in the publication when you use `{%pk%}`.
- For update conditions that compare against previous values, configure Postgres replica identity so Postgres sends the required previous columns.
- Redis `WRONGTYPE`, unknown command, and syntax errors are treated as non-retryable downstream errors.
- Multiple commands for a single row are atomic in Redis because they are wrapped in `MULTI` and `EXEC`.
- Batches are flushed by size (`flushBufferSize`) or time (`flushInterval`), whichever happens first. See [Redis flush pipeline](config.md#redis-flush-pipeline) for how batch size, queue depth, and workers interact.
