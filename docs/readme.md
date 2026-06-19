# pg2redis

**pg2redis** is a PostgreSQL-to-Redis streamer. It reads row-level changes from PostgreSQL logical replication and applies configured Redis commands for inserts, updates, deletes, and initial snapshots.

---

## Features

- Real-time Redis updates from PostgreSQL WAL.
- Configurable Redis writes with templates for keys, values, hashes, streams, sorted sets, sets, and pub/sub.
- Operation-specific command mappings for insert, update, and delete.
- Conditional command execution based on current or previous row values.
- Initial snapshots for loading existing table data before streaming starts.
- PostgreSQL publication and replication slot management when the application owns replication setup.
- At-least-once processing with saved LSN state in Redis.

---

## Installation

### From Binary

Use the binary provided by pgwalk for your target platform.

### From Docker Hub

Docker Hub URL: `https://hub.docker.com/r/alikpgwalk/pg2redis`

Image name: `alikpgwalk/pg2redis`

Run it with your configuration mounted into the container:

```bash
docker pull alikpgwalk/pg2redis:latest
docker run --rm \
  -e PGWALK_LIC_PG2REDIS=<license-key> \
  -v "$PWD/config.yaml:/etc/pg2redis/config.yaml:ro" \
  alikpgwalk/pg2redis:latest
```

The application loads configuration from `/etc/pg2redis/config.yaml`, `./config.yaml`, and the file named by `PG2REDIS_CONFIG_PATH`.

---

## Configuration

You can configure the app via:

- YAML configuration file
- Environment variables

Environment variables override YAML values.

See [Configuration](config.md) for the full reference.

---

## Redis Commands

pg2redis maps each database row change to one or more Redis commands. A table can use a single default command, operation-specific commands, conditional commands, or a multi-command group.

See [Redis commands](redis-commands.md) for supported commands, templates, conditions, and examples.

---

## Snapshots

Snapshots load existing table data into Redis before logical replication streaming begins. Snapshot progress is tracked in PostgreSQL and the completed snapshot LSN is stored in Redis.

See [Snapshots](snapshot.md) for modes, mechanics, state tables, and operational notes.

---

## PostgreSQL Replication

pg2redis uses PostgreSQL logical replication with the built-in `pgoutput` plugin.

See [Replication management](pgrepl.md) for PostgreSQL setup, slots, publications, failover notes, and permissions.

---

## License

pg2redis requires a valid pgwalk license.

Set the license key directly:

```bash
PGWALK_LIC_PG2REDIS=<license key>
```

Or save the key to a file and point pg2redis at it:

```bash
PGWALK_LICFILE_PG2REDIS=/path/to/key.pg2redislic
```

The application will not start without a valid license.
