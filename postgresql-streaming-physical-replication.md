# How to Set Up Streaming Physical Replication with PostgreSQL

*Author: Rijavan Inamdar*

---

## Introduction

This document provides step-by-step instructions for configuring **streaming (physical) replication** with PostgreSQL — considered the most reliable replication method for achieving High Availability.

Almost all HA solutions for PostgreSQL leverage built-in streaming replication. In addition to streaming replication, PostgreSQL also supports logical replication, and there are external trigger-based replication solutions like Slony, Bucardo, and Londiste. Among all replication types, streaming replication is the most lightweight and reliable, with the least delay — it's always recommended for critical production database environments.

## Table of Contents

- [Restrictions and Behavior of Physical Replication](#restrictions-and-behavior-of-physical-replication)
- [Functionalities](#functionalities)
- [Steps to Set Up Streaming Replication](#steps-to-set-up-streaming-replication-between-a-primary-and-one-standby)
- [Optional: Standby Using Replication Slots](#optional-standby-using-replication-slots)
- [Validating That Replication Is Set Up](#validating-that-replication-is-set-up)
- [Checking Replication Lag](#checking-replication-lag-between-the-primary-and-the-standby)
- [Failover to Standby](#failover-to-standby)
- [Additional References](#additional-references)
- [Tested Versions](#tested-versions)

---

## Restrictions and Behavior of Physical Replication

Before setting up streaming replication, keep the following in mind:

- Both the primary and its standby(s) must run the **same PostgreSQL major version** — streaming replication across major versions is not possible.
- A primary can have **multiple standby replicas**.
- Standbys accept **read-only queries** if `hot_standby` is set to `on`. A standby never accepts write operations (DMLs or DDLs).
- **Replication filters are not possible** in streaming/physical replication — the entire PostgreSQL instance is replicated. Conceptually, a standby's data directory is an exact copy of the primary's.
- There can be **no difference in users, roles, or privileges** between a primary and its standby. Any user needed on the standby must be created on the primary and replicated over.
- `max_connections` on the standby must be **equal to or higher** than on the primary.
- You cannot make changes directly on a standby, aside from a few parameter changes. If something like index or table corruption needs fixing on a standby, the fix must be applied on the **primary** (e.g., drop and recreate the corrupted index there) — the change then replicates automatically.

## Functionalities

- **`UNLOGGED` tables are not replicated** to the standby. A table can be converted using:
  ```sql
  ALTER TABLE table_name SET UNLOGGED;
  ```
  Note that unlogged tables are not crash-safe — they're truncated on an instance crash.
- If `max_connections` needs to increase on the primary, the standby's value must be **increased first**, and the standby restarted, before applying the change on the primary.
- A primary and its standbys **can have different `pg_hba.conf` settings** for authentication.
- To reduce the chance of query cancellation on the standby due to a primary-side vacuum or DDL, tune `hot_standby_feedback`, `max_standby_streaming_delay`, and `max_standby_archive_delay` appropriately.
- `VACUUM` / `VACUUM FULL` / `REINDEX` or other maintenance never needs to run on the standby directly — these operations on the primary are replicated automatically.
- **Cascaded streaming replication** is supported from PostgreSQL 9.2 onward — a standby can itself replicate to another standby.
- Streaming replication depends entirely on WALs (transaction logs). Ensure sufficient WAL retention on the primary, and have a WAL archiving strategy so standbys can fetch WALs they've missed.
- For each standby, the primary starts one **WAL Sender** process; each standby runs one **WAL Receiver** process. Streaming replication is single-threaded — one sender per standby, one receiver per standby.
- All tablespace mount points used on the primary must also **exist on the standby**.

---

## Steps to Set Up Streaming Replication Between a Primary and One Standby

### Step 1 — Create a Replication User

Create a user on the primary with `REPLICATION` and `LOGIN` privileges — only a user with the `REPLICATION` attribute can initiate a streaming replication connection:

```sql
CREATE USER replicator
WITH REPLICATION
ENCRYPTED PASSWORD 'replicator';
```

### Step 2 — Configure Mandatory Parameters on the Primary

| Parameter | Purpose |
|---|---|
| `archive_mode` | Must be `ON` to enable WAL archiving |
| `wal_level` | Must be `replica` |
| `max_wal_senders` | Minimum of 3 for a single standby — allow at least 2 senders per streaming standby, plus one spare in case of disconnection |
| `wal_keep_segments` / `wal_keep_size` | Controls how much WAL is retained in `pg_wal`. Each WAL segment is 16MB by default. Start with 100+ depending on available space and WAL generation rate. (`wal_keep_segments` was replaced by `wal_keep_size` starting in PostgreSQL 13.) Set high enough that logs aren't removed before backup operations complete. |
| `archive_command` | A shell command or script used to archive WAL segments — anything from a simple copy command to logic that ships WALs to cloud storage or a backup server |
| `listen_addresses` | Set to `*` or a comma-separated list of interface IPs to listen on |
| `hot_standby` | Must be `ON` on the standby (has no effect on the primary, though it's copied over automatically) — enables read-only queries on the standby |
| `max_replication_slots` | Should allow enough slots if replication slots will be used (optional) |

**Example configuration commands** (some of these may already be default in recent PostgreSQL versions):

```sql
ALTER SYSTEM SET archive_mode TO 'ON';
ALTER SYSTEM SET wal_level TO 'replica';               -- default in recent releases
ALTER SYSTEM SET max_wal_senders TO '5';                -- defaults to 10 in newer versions
ALTER SYSTEM SET wal_keep_segments TO '10';              -- for versions before PG13
ALTER SYSTEM SET archive_command TO 'test ! -f /mnt/server/archivedir/%f && cp %p /mnt/server/archivedir/%f';
ALTER SYSTEM SET listen_addresses TO '*';
ALTER SYSTEM SET hot_standby TO 'ON';                    -- defaults to ON in newer releases
```

A restart is required for some parameters (like `archive_mode`) to take effect:

```bash
pg_ctl -D $PGDATA restart -mf
```

### Step 3 — Update `pg_hba.conf` on the Primary

Add an entry allowing replication connections from the standby. The default location of `pg_hba.conf` is the data directory (though this can be overridden in `postgresql.conf`). On Ubuntu/Debian, it's located alongside `postgresql.conf`.

```bash
vi pg_hba.conf
```

Add the following line to the end of the file:

```
host replication replicator 192.168.0.28/32 md5
```

> The CIDR `192.168.0.28/32` above is just an example — it should match your standby's IP address(es).

Reload the configuration to apply the change:

```bash
pg_ctl -D $PGDATA reload
```

Or:

```bash
psql -U postgres -p 5432 -c "select pg_reload_conf()"
```

### Step 4 — Take a Base Backup and Stream It to the Standby

Take a "plain" format backup of the primary using `pg_basebackup`, run **from the standby**, so the backup is stored there as an exact copy of the primary's data directory. Alternatively, a "tar" format backup can be taken on the primary and copied to the standby. For more detail, see the PostgreSQL documentation on [`pg_basebackup`](https://www.postgresql.org/docs/current/app-pgbasebackup.html).

A plain-format backup can be streamed directly from the primary to the standby over a PostgreSQL connection:

```bash
pg_basebackup -h primary_host -U replicator -p 5432 -D $PGDATA -Fp -P -Xs -R
```

The `-R` flag automatically creates a `standby.signal` file and writes the standby-related parameters into `postgresql.auto.conf`. A `standby.signal` file is **mandatory** for streaming replication to work.

> If a tar-format backup is used instead of the plain-format example above, `standby.signal` must be created **manually**.

**Example `postgresql.auto.conf`** (PostgreSQL 12 and above):

```ini
# Do not edit this file manually!
# It will be overwritten by the ALTER SYSTEM command.
max_wal_senders = '5'
archive_command = 'test ! -f /mnt/server/archivedir/%f && cp %p /mnt/server/archivedir/%f'
listen_addresses = '*'
hot_standby = 'ON'
archive_mode = 'ON'
wal_level = 'replica'
primary_conninfo = 'user=replicator password=replicator channel_binding=prefer host=192.114.29.185 port=5432 sslmode=prefer sslcompression=0 sslsni=1 ssl_min_protocol_version=TLSv1.2 gssencmode=prefer krbsrvname=postgres target_session_attrs=any'
```

Connection details to the primary are set via `primary_conninfo` — this is populated automatically when using `-R` with `pg_basebackup`.

> You can also specify `archive_cleanup_command` to remove WALs from the archive location once no longer needed by any standby:
> ```ini
> archive_cleanup_command = 'pg_archivecleanup /path/to/archive %r'
> ```
> **We don't recommend automatic archive cleanup** — WAL segment files are essential for PITR, and WAL retention should be a deliberate part of your backup strategy, not something automatically pruned.

### Step 5 — Start the Standby Instance

If a **plain-format** backup was used (Step 4), all files are already copied into the standby's data directory — backup and restore effectively happen in one step.

If a **tar-format** backup was used instead, you'll need to manually unpack it into the standby's data directory and (on PostgreSQL versions before 12) create a `recovery.conf` file, as described above.

Once ready, start the standby:

```bash
pg_ctl -D $PGDATA start
```

### Step 6 — Configure `restore_command` (Recommended for Production)

In production, it's always advisable to set `restore_command` appropriately. This parameter takes a shell command or script used to fetch a WAL file the standby needs, if it's no longer available to stream directly from the primary.

For example, if a network interruption cuts off replication for an extended period, the primary may have already discarded the WAL files generated during that window. Archiving WALs to a safe location — with `restore_command` configured to retrieve them — protects against this.

On PostgreSQL 12+, set this directly as a regular parameter (rather than in a `recovery.conf` file, which was used on older versions):

```ini
restore_command = 'cp /mnt/server/archivedir/%f "%p"'
```

> **Note:** On versions before PostgreSQL 12, this setting lived in the standby's `recovery.conf` file instead.
>
> ⚠️ **Important:** Changing `restore_command` cannot be applied online — it requires a PostgreSQL restart to take effect.

---

## Optional: Standby Using Replication Slots

PostgreSQL supports an optional feature allowing a standby to use a **replication slot**. A replication slot prevents the primary from removing WAL segments still required by the standby assigned to that slot.

Create a physical replication slot on the primary:

```sql
SELECT pg_create_physical_replication_slot('slot1');
```

```
 pg_create_physical_replication_slot
-------------------------------------
 (slot1,)
(1 row)
```

Specify the slot name when taking the backup used to build the standby:

```bash
pg_basebackup -U replicator -p 5432 -S slot1 -D $PGDATA -P -Xs -R --checkpoint=fast -h primary_host
```

The slot name is added automatically to the standby's configuration:

```bash
cat postgresql.auto.conf
```

```ini
primary_conninfo = 'user=replicator password=****** channel_binding=prefer host=pg0 port=5432 sslmode=prefer sslcompression=0 sslsni=1 ssl_min_protocol_version=TLSv1.2 gssencmode=prefer krbsrvname=postgres target_session_attrs=any'
primary_slot_name = 'slot1'
```

> A slot can also be added later by updating this parameter after the fact.

Once the standby connects using the replication slot, it becomes visible on the primary via `pg_replication_slots`:

```sql
SELECT * FROM pg_replication_slots;
```

```
 slot_name | plugin | slot_type | datoid | database | temporary | active | active_pid | xmin | catalog_xmin | restart_lsn | confirmed_flush_lsn | wal_status | safe_wal_size | two_phase
-----------+--------+-----------+--------+----------+-----------+--------+------------+------+--------------+-------------+---------------------+------------+---------------+-----------
 slot1     |        | physical  |        |          | f         | t      |        920 |      |              | 1/5B000148  |                     | reserved   |               | f
(1 row)
```

---

## Validating That Replication Is Set Up

### 1. Check for the WAL Sender and WAL Receiver Processes

**On the primary:**

```bash
ps -eaf | grep sender
```

**On the standby:**

```bash
ps -eaf | grep receiver
```

You should see the relevant process running on each side.

### 2. Query Replication Status

**On the primary:**

```sql
SELECT * FROM pg_stat_replication;
```

**On the standby:**

```sql
SELECT * FROM pg_stat_wal_receiver;
```

---

## Checking Replication Lag Between the Primary and the Standby

Run the following on the primary to calculate the lag in bytes:

```sql
SELECT application_name, client_addr, state, sent_offset - (replay_offset - (sent_lsn - replay_lsn) * 255 * 16 ^ 6 ) AS byte_lag
   FROM ( SELECT application_name, client_addr, client_hostname, state,
   ('x' || lpad(split_part(sent_lsn::TEXT,   '/', 1), 8, '0'))::bit(32)::bigint AS sent_lsn,
   ('x' || lpad(split_part(replay_lsn::TEXT, '/', 1), 8, '0'))::bit(32)::bigint AS replay_lsn,
   ('x' || lpad(split_part(sent_lsn::TEXT,   '/', 2), 8, '0'))::bit(32)::bigint AS sent_offset,
   ('x' || lpad(split_part(replay_lsn::TEXT, '/', 2), 8, '0'))::bit(32)::bigint AS replay_offset
   FROM pg_stat_replication ) AS s;
```

**Example output:**

```
 application_name | client_addr   |   state   | byte_lag
------------------+---------------+-----------+----------
 walreceiver      | 192.114.29.36 | streaming |        0
(1 row)
```

---

## Failover to Standby

To promote a standby to primary so it can begin accepting write workloads, run the following on the standby:

```bash
pg_ctl -D $PGDATA promote
```

> ⚠️ **Important:** Redirect all application traffic to the standby **before** performing the failover.

---

## Additional References

- [WAL Configuration — PostgreSQL Documentation](https://www.postgresql.org/docs/current/static/wal-configuration.html)
- [Streaming Replication Protocol — PostgreSQL Documentation](https://www.postgresql.org/docs/current/static/protocol-replication.html)
- [System Administration Functions — PostgreSQL Documentation](https://www.postgresql.org/docs/current/functions-admin.html)
- [Hot Standby — PostgreSQL Documentation](https://www.postgresql.org/docs/current/hot-standby.html)

---

## Tested Versions

This article has been tested against the following versions:

| Component | Version |
|---|---|
| PostgreSQL | 12, 13, 14, 15, 16 |
