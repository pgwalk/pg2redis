# pg2redis

**pg2redis** is a Postgres-to-Redis streamer. It reads row-level changes from Postgres logical replication and applies configured Redis commands for inserts, updates, deletes, and initial snapshots.

---

## Features

- Real-time Redis updates from Postgres WAL.
- Configurable Redis writes with templates for keys, values, hashes, streams, sorted sets, sets, and pub/sub.
- Operation-specific command mappings for insert, update, and delete.
- Conditional command execution based on current or previous row values.
- Initial snapshots for loading existing table data before streaming starts.
- Postgres publication and replication slot management when the application owns replication setup.
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

See [Configuration](docs/config.md) for the full reference.

---

## Redis Commands

pg2redis maps each database row change to one or more Redis commands. A table can use a single default command, operation-specific commands, conditional commands, or a multi-command group.

See [Redis commands](docs/redis-commands.md) for supported commands, templates, conditions, and examples.

---

## Snapshots

Snapshots load existing table data into Redis before logical replication streaming begins. Snapshot progress is tracked in Postgres and the completed snapshot LSN is stored in Redis.

See [Snapshots](docs/snapshot.md) for modes, mechanics, state tables, and operational notes.

---

## Postgres Replication

pg2redis uses Postgres logical replication with the built-in `pgoutput` plugin.

See [Replication management](docs/pgrepl.md) for Postgres setup, slots, publications, failover notes, and permissions.

---

## License

pg2redis requires a valid pgwalk license.

Current license key, valid until `2027-01-01`:

```bash
PGWALK_LIC_PG2REDIS=p7WCiwxK4mvAHZve4YavXc9ip9q03kFmTQe2Qsy8TCGcv8FdakqSFh9z74pw5bfryloykJaA7A826hF38X5Q60v3W05uH3hoAhtVXCmBn7kc1BqotzhjF6aNzOva7MlElTsCcRpwQEeUBlsczo8O0vozUFAxy5eH3zRHcN938wuDTT4DjfAOABcBJRmlWjSdERq
```

Set the license key directly:

```bash
PGWALK_LIC_PG2REDIS=<license key>
```

Or save the key to a file and point pg2redis at it:

```bash
PGWALK_LICFILE_PG2REDIS=/path/to/key.pg2redislic
```

The application will not start without a valid license.
