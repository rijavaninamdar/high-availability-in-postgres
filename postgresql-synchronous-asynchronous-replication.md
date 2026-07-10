# Synchronous and Asynchronous Streaming Replication in PostgreSQL

*Author: Rijavan Inamdar*

---

## Introduction

PostgreSQL streaming replication is **asynchronous by default**. In asynchronous replication, WALs are shipped to standbys *after* a transaction commit. This can lead to data loss during a failover, since committed transactions may not yet have been shipped to the standby.

**Synchronous commit** addresses this for situations where data loss is unacceptable: every write transaction waits for confirmation that the commit has been written to the transaction log on the disks of both the primary *and* the standby. This comes at a performance cost, since every write transaction waits for the standby's confirmation — and can fail outright if it can't write to the standby.

## Table of Contents

- [Parameters Used to Configure Synchronous Replication](#parameters-used-to-configure-synchronous-replication)
- [Setting Up Synchronous Replication](#setting-up-synchronous-replication)
- [Asynchronous Replication](#asynchronous-replication)

---

## Parameters Used to Configure Synchronous Replication

### `synchronous_commit`

Controls how strictly a commit waits for standby confirmation. Acceptable values are `on`, `remote_write`, `local`, and `off`. The default is `on`, which is the safest setting.

To enable synchronous replication, set this to either `on` or `remote_write`:

| Value | Guarantee |
|---|---|
| `on` | The standby has received the commit record **and flushed it to disk**. Provides the strongest guarantee against transaction loss. |
| `remote_write` | The standby has received the commit record **and written it out to its operating system** (but not necessarily flushed to disk). Reduces latency, but an OS crash on the standby could still lose the transaction. |

See the [PostgreSQL documentation on WAL configuration](https://www.postgresql.org/docs/current/runtime-config-wal.html) for full details.

### `synchronous_standby_names`

A list of standby names that must remain synchronous with the primary. The "name" here refers to the `application_name` visible when querying `pg_stat_replication` on the primary (the default is `walreceiver`).

- For **streaming replication**, set `application_name` via the `primary_conninfo` parameter on the standby.
- For **logical replication**, the subscription name is treated as the `application_name` — so `synchronous_standby_names` should reference the subscription's name instead.

**Behavior by value:**

- **`on`** — Commits wait until the current synchronous standby confirms it has received the commit record and flushed it to disk. This ensures the transaction survives unless both the primary and the standby suffer storage corruption.
- **`remote_write`** — Commits wait until the current synchronous standby confirms it has received the commit record and written it to its OS — but the data hasn't necessarily reached stable storage on the standby yet.

---

## Setting Up Synchronous Replication

### Step 1 — Set Up Base Replication

Before configuring synchronous mode, set up:

- Streaming (physical) replication between a primary and a standby.

### Step 2 — Set `application_name` on the Standby (Streaming Replication Only)

Add `application_name` to the `primary_conninfo` parameter:

```ini
primary_conninfo = 'user=replicator password=replicator application_name=standby1 host=192.168.0.11 port=5432'
```

Restart the standby to apply:

```bash
pg_ctl -D $PGDATA restart -mf
```

> **Note:** `recovery.conf` was deprecated in PostgreSQL 12. On PostgreSQL 12 and newer, set `primary_conninfo` directly in `postgresql.conf` instead.
>
> For **logical replication**, no changes are needed on the subscriber node for this step.

### Step 3 — Configure Synchronous Parameters on the Primary

```ini
synchronous_commit = ON
synchronous_standby_names = 'application_name'
```

Substitute `application_name` with the value used in `primary_conninfo` (streaming replication) or the subscription name (logical replication).

Reload the configuration on the primary (or publishing node):

```bash
pg_ctl -D $PGDATA reload
```

### Step 4 — Validate the Setup

Query `pg_stat_replication` on the primary and confirm `sync_state` shows `sync`:

```sql
SELECT * FROM pg_stat_replication;
```

```
-[ RECORD 1 ]----+------------------------------
pid              | 2439
usesysid         | 16392
usename          | replicator
application_name | standby1
client_addr      | 192.168.0.12
client_hostname  |
client_port      | 59480
backend_start    | 2018-06-07 14:58:14.855046-04
backend_xmin     |
state            | streaming
sent_lsn         | 0/6000140
write_lsn        | 0/6000140
flush_lsn        | 0/6000140
replay_lsn       | 0/6000140
write_lag        |
flush_lag        |
replay_lag       |
sync_priority    | 1
sync_state       | sync
```

---

## Asynchronous Replication

If `synchronous_standby_names` is left empty or unset on the primary/publisher node, replication automatically operates in **asynchronous** mode — the default behavior described in the introduction.
