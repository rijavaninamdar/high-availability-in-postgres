# How to Set Up Cascade Replication in PostgreSQL

*Author: Rijavan Inamdar*

---

## Introduction

Setting up a PostgreSQL replica with streaming replication is straightforward — that process is covered in [How to Set Up Streaming Physical Replication with PostgreSQL](#).

Beyond the most common form of replicating directly from the primary, PostgreSQL also supports streaming replication using a **standby server** as the source. This topology is known as **cascaded replication**:

```
Primary --> Standby 1 --> Cascaded Standby 2
```

This approach is useful when you need multiple standbys and want to avoid overloading the primary with WAL streaming to every one of them. If the first-layer standby is located close to the primary, and there are no conflicting sessions, its performance is nearly as fast as the primary's — making it an efficient relay point for additional standbys.

## Table of Contents

- [Creating Standbys](#creating-standbys)
- [Creating a Cascaded Standby](#creating-a-cascaded-standby)
- [Node Failure (High Availability) Considerations](#node-failure-high-availability-considerations)
- [Important Points to Remember](#important-points-to-remember)

---

## Creating Standbys

This article uses the following setup for illustration:

| Role | Node |
|---|---|
| Primary Server | `pg0` |
| First-Layer Standby | `pg1` |
| Second-Layer (Cascaded) Standby | `pg2` |

The overall replication topology is: **`pg0 → pg1 → pg2`**

### Creating the First-Layer Standby

Creating a first-layer standby follows the same process as a regular standby (see the linked article above). Take a data directory copy of `pg0` onto `pg1` using `pg_basebackup`, run from `pg1`:

```bash
pg_basebackup -U replicator -p 5432 -D $PGDATA -P -Xs -R --checkpoint=fast -h pg0
```

Start the `pg1` instance for streaming replication.

### Verify Replication Status on the Primary

```sql
SELECT * FROM pg_stat_replication;
```

```
-[ RECORD 1 ]----+------------------------------
pid              | 12712
usesysid         | 16406
usename          | replicator
application_name | walreceiver
client_addr      | 192.168.50.20
client_hostname  |
client_port      | 53380
backend_start    | 2021-11-08 06:04:20.312135+00
backend_xmin     |
state            | streaming
sent_lsn         | 0/B014878
write_lsn        | 0/B014878
flush_lsn        | 0/B014878
replay_lsn       | 0/B014878
write_lag        |
flush_lag        |
replay_lag       |
sync_priority    | 0
sync_state       | async
reply_time       | 2021-11-08 11:37:22.855555+00
```

The IP address `192.168.50.20` belongs to `pg1`, confirming it's streaming from `pg0`.

---

## Creating a Cascaded Standby

For the cascaded standby, the backup can be taken from either the primary or the standby. In this example, the backup is taken **from the first-level standby (`pg1`)** to build the new cascaded standby (`pg2`).

Run `pg_basebackup` from `pg2`, targeting `pg1`:

```bash
pg_basebackup -U replicator -p 5432 -D $PGDATA -P -Xs -R --checkpoint=fast -h pg1
```

Because the backup was taken from the standby, `primary_conninfo` will point to the standby node. On PostgreSQL 12+, this is recorded in `postgresql.auto.conf`.

### Example: `postgresql.auto.conf` on the Cascaded Standby

```ini
primary_conninfo = 'user=replicator password=vagrant channel_binding=prefer host=pg0 port=5432 sslmode=prefer sslcompression=0 ssl_min_protocol_version=TLSv1.2 gssencmode=prefer krbsrvname=postgres target_session_attrs=any'
primary_conninfo = 'user=replicator password=vagrant channel_binding=prefer host=pg1 port=5432 sslmode=prefer sslcompression=0 ssl_min_protocol_version=TLSv1.2 gssencmode=prefer krbsrvname=postgres target_session_attrs=any'
```

There are **two entries** here — the first pointing to the primary, the second to the standby. Since the same parameter is set twice, **the second (last) value takes effect**.

> **Tip:** Keeping both connection strings around has a practical advantage — the cascaded standby can be redirected to connect directly to the primary instead of the first-level standby at any time. This required a restart through PostgreSQL 12; from **PostgreSQL 13 onward, only a reload is needed** — no restart required.

### Verify the Cascaded Connection on the First-Level Standby

```sql
SELECT * FROM pg_stat_replication;
```

```
-[ RECORD 1 ]----+------------------------------
pid              | 8619
usesysid         | 16406
usename          | replicator
application_name | walreceiver
client_addr      | 192.168.50.30
client_hostname  |
client_port      | 44136
backend_start    | 2021-11-08 12:23:52.947322+00
backend_xmin     |
state            | streaming
sent_lsn         | 0/B014878
write_lsn        | 0/B014878
flush_lsn        | 0/B014878
replay_lsn       | 0/B014878
write_lag        |
flush_lag        |
replay_lag       |
sync_priority    | 0
sync_state       | async
reply_time       | 2021-11-08 12:24:03.021639+00
```

Here, the client IP `192.168.50.30` belongs to the cascaded standby.

---

## Node Failure (High Availability) Considerations

### Loss of the Primary Server (`pg0`)

If the primary is lost, the first-level standby (`pg1`) can be promoted to become the new primary. The cascaded standby (`pg2`) continues replicating from `pg1` without any change to its `primary_conninfo`, since it was already configured to connect there.

> Make sure `recovery_target_timeline` is set to `latest` on the cascaded standby to ensure it correctly follows the new primary's timeline after promotion.

### Loss of the Standby Server (`pg1`)

If the first-level standby is lost, `primary_conninfo` on the cascaded standby can be updated to point directly to the primary. This change required a restart through PostgreSQL 12.

---

## Important Points to Remember

### 1. WAL Retention

Since WAL segments are streamed from the standby to the cascaded standby, it's important to ensure **sufficient WAL retention on the standby side as well** — not just the primary. We recommend setting a sufficiently high `max_wal_size` on the standby to retain several hours' worth of WAL segments. A shared WAL archive directory is also a solid alternative.

### 2. Conflicts on the Standby

Conflicting statements or sessions on the intermediate standby **do not affect the cascaded standby**. Recovery on the cascaded standby (`pg2`) is independent of recovery on the intermediate standby (`pg1`) — this is because no WAL is generated on a standby; all WAL originates from the primary.
