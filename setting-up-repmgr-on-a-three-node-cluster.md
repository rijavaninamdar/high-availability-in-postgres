# Setting Up repmgr on a Three-Node Cluster

## Introduction

`repmgr` is one of the oldest High Availability (HA) frameworks for PostgreSQL and was very popular a few years back. There are still a huge number of installations of repmgr today, and many organizations — especially those that migrated to PostgreSQL earlier — continue to use it as their standard practice.

Some well-known drawbacks of repmgr include:

- Outdated architecture compared to Raft-based algorithms
- Poor community participation in development — mainly a single-vendor solution
- Requires a library to be loaded into PostgreSQL (invasive approach)
- Requires schema and objects to be created in the database
- Needs an additional witness server
- Lacks full protection from split-brain conditions

This article describes the steps involved in installing and configuring repmgr on a 3-node cluster.

> If you're new to repmgr, we'd recommend reading a quick introduction to repmgr before proceeding.

**Environment used in this guide:**

| Component | Version |
|-----------|---------|
| PostgreSQL | 13 |
| repmgr | 5.2.1 |
| OS | CentOS 7.9 |

Similar steps should work for other environments and versions as well.

This illustration uses three host machines — `pg0`, `pg1`, and `pg2` — all connected over the network and reachable from one another on both the SSH port and the PostgreSQL port.

## Process

### 1. Install the repmgr software

Unlike HA solutions such as Patroni, the repmgr package and installation are PostgreSQL version-specific. The PGDG repository provides packages specific to each PostgreSQL version, which need to be installed on the primary and standby(s).

```bash
$ sudo yum search repmgr
Last metadata expiration check: 2:35:11 ago on Mon 14 Sep 2020 09:05:50 AM UTC.
================================== Name & Summary Matched: repmgr ==================================
repmgr12-devel.x86_64 : Development header files of repmgr
repmgr11-devel.x86_64 : Development header files of repmgr
======================================= Name Matched: repmgr =======================================
repmgr12.x86_64 : Replication Manager for PostgreSQL Clusters
repmgr11.x86_64 : Replication Manager for PostgreSQL Clusters
repmgr10.x86_64 : Replication Manager for PostgreSQL Clusters
repmgr96.x86_64 : Replication Manager for PostgreSQL Clusters
repmgr95.x86_64 : Replication Manager for PostgreSQL Clusters
```

Installation is straightforward using `yum` or `dnf`:

```bash
[postgres@c82pri ~]$ sudo yum install repmgr13 --nogpgcheck
Last metadata expiration check: 2:28:35 ago on Mon 14 Sep 2020 09:05:50 AM UTC.
Dependencies Resolved

=======================================================================================================================================
 Package                         Arch                         Version                               Repository                    Size
=======================================================================================================================================
Installing:
 repmgr_13                       x86_64                       5.2.1-1.rhel7                         pgdg13                       284 k

Transaction Summary
=======================================================================================================================================
Install  1 Package

Total download size: 284 k
Installed size: 1.0 M
```

Install `rsync`:

```bash
sudo yum install -y rsync
```

> The same packages need to be installed on the standby server(s) as well.

### 2. Configure PostgreSQL for repmgr

repmgr stores its metadata in its own database schema, so it's highly recommended to have a dedicated database and user account:

```sql
CREATE USER repmgr WITH SUPERUSER PASSWORD 'secret';
ALTER USER repmgr SET search_path TO repmgr, "$user", public;
CREATE DATABASE repmgr OWNER repmgr;
```

Add the extension on all nodes (primary and standby(s)):

```sql
ALTER SYSTEM SET shared_preload_libraries = repmgr;
```

> **Tip:** Check whether any other libraries are already specified in `shared_preload_libraries` and add `repmgr` to that existing list rather than overwriting it.

Make sure WAL archiving is enabled:

```sql
ALTER SYSTEM SET archive_mode = on;
```

Ensure the `archive_command` is also set to move WAL archives to a backup location. If nothing is ready yet, you can temporarily stub the archiver process:

```sql
ALTER SYSTEM SET archive_command = '/bin/true';
```

It's good practice to have sufficient WAL retention:

```sql
ALTER SYSTEM SET max_wal_size = '10GB';
ALTER SYSTEM SET min_wal_size = '4GB';
ALTER SYSTEM SET wal_keep_size = '2GB';
```

`pg_hba.conf` should allow repmgr connections from both local and remote servers:

```
local   replication   repmgr                              trust
host    replication   repmgr      127.0.0.1/32            trust
host    replication   repmgr      192.168.50.0/24         trust

local   repmgr        repmgr                              trust
host    repmgr        repmgr      127.0.0.1/32            trust
host    repmgr        repmgr      192.168.50.0/24         trust
```

Restart the PostgreSQL instance to load the repmgr libraries:

```bash
$ sudo systemctl restart postgresql-13
```

> The exact command above may vary depending on your Linux distribution.

### 3. Configure host machines for repmgr

**Configure password-less SSH between nodes**

The switchover operation requires actions to be performed on different nodes. repmgr uses password-less SSH to achieve this — copy the public key of the SSH key-pair to `~/.ssh/authorized_keys` on the other machine(s). Verify that password-less SSH works before proceeding.

**Verify `rsync`**

Make sure the host machine has the `rsync` utility available:

```bash
$ rsync --version
rsync  version 3.1.3  protocol version 31
...
```

**Clean the standby-side data directory**

There should be no PostgreSQL instance on the standby side, but the data directory must exist, be clean, and be owned by the `postgres` user:

```bash
sudo systemctl stop postgresql-13
rm -rf $PGDATA/*
```

**Verify standby-to-primary connectivity**

Connectivity over the PostgreSQL network protocol must be confirmed before proceeding, since repmgr uses physical standby streaming:

```bash
psql 'host=pg0 user=repmgr dbname=repmgr connect_timeout=2'
```

### 4. Configure repmgr

repmgr's configuration is located in `/etc/repmgr/`. For example, the configuration file for repmgr on PostgreSQL 13 will be located at `/etc/repmgr/13/repmgr.conf`. Primary and standby nodes each need a `repmgr.conf`.

**Primary node (`pg0`):**

```bash
sudo vi /etc/repmgr/13/repmgr.conf
```

```ini
node_id=1
node_name='pg0'
conninfo='host=pg0 user=repmgr dbname=repmgr connect_timeout=2'
data_directory='/var/lib/pgsql/13/data'
```

**First standby (`pg1`):**

```ini
node_id=2
node_name='pg1'
conninfo='host=pg1 user=repmgr dbname=repmgr connect_timeout=2'
data_directory='/var/lib/pgsql/13/data'
```

**Second standby (`pg2`):**

```ini
node_id=3
node_name='pg2'
conninfo='host=pg2 user=repmgr dbname=repmgr connect_timeout=2'
data_directory='/var/lib/pgsql/13/data'
```

### 5. Register the primary node

Registration needs to be performed as the `postgres` OS user:

```bash
$ /usr/pgsql-13/bin/repmgr -f /etc/repmgr/13/repmgr.conf primary register
INFO: connecting to primary database...
NOTICE: attempting to install extension "repmgr"
NOTICE: "repmgr" extension successfully installed
NOTICE: primary node record (ID: 1) registered
```

Verify the status:

```bash
$ /usr/pgsql-13/bin/repmgr -f /etc/repmgr/13/repmgr.conf cluster show
ID | Name | Role    | Status    | Upstream | Location | Priority | Timeline | Connection string
---+------+---------+-----------+----------+----------+----------+----------+----------------------------------------------------------------
1  | pg0  | primary | * running |          | default  | 100      | 1        | host=192.168.50.10 user=repmgr dbname=repmgr connect_timeout=2
```

> Standby nodes can be registered in a similar way later.

### 6. Clone the standby from the primary

Run the following commands on the standby side. First, perform a dry run:

```bash
/usr/pgsql-13/bin/repmgr -h pg0 -U repmgr -d repmgr -f /etc/repmgr/13/repmgr.conf standby clone --dry-run
```

```text
NOTICE: destination directory "/var/lib/pgsql/13/data" provided
INFO: connecting to source node
DETAIL: connection string is: host=pg0 user=repmgr dbname=repmgr
DETAIL: current installation size is 32 MB
INFO: "repmgr" extension is installed in database "repmgr"
INFO: replication slot usage not requested; no replication slot will be set up for this standby
INFO: parameter "max_wal_senders" set to 10
NOTICE: checking for available walsenders on the source node (2 required)
INFO: sufficient walsenders available on the source node
DETAIL: 2 required, 10 available
NOTICE: checking replication connections can be made to the source server (2 required)
INFO: required number of replication connections could be made to the source server
DETAIL: 2 replication connections required
NOTICE: standby will attach to upstream node 1
HINT: consider using the -c/--fast-checkpoint option
INFO: all prerequisites for "standby clone" are met
```

Once the dry run passes, perform the actual clone:

```bash
$ /usr/pgsql-13/bin/repmgr -h pg0 -U repmgr -d repmgr -f /etc/repmgr/13/repmgr.conf standby clone
```

```text
NOTICE: destination directory "/var/lib/pgsql/13/data" provided
INFO: connecting to source node
DETAIL: connection string is: host=192.168.50.10 user=repmgr dbname=repmgr
DETAIL: current installation size is 31 MB
INFO: replication slot usage not requested; no replication slot will be set up for this standby
NOTICE: checking for available walsenders on the source node (2 required)
NOTICE: checking replication connections can be made to the source server (2 required)
INFO: creating directory "/var/lib/pgsql/13/data"...
NOTICE: starting backup (using pg_basebackup)...
HINT: this may take some time; consider using the -c/--fast-checkpoint option
INFO: executing:
  pg_basebackup -l "repmgr base backup"  -D /var/lib/pgsql/13/data -h 192.168.50.10 -p 5432 -U repmgr -X stream
NOTICE: standby clone (using pg_basebackup) complete
NOTICE: you can now start your PostgreSQL server
HINT: for example: pg_ctl -D /var/lib/pgsql/13/data start
HINT: after starting the server, you need to register this standby with "repmgr standby register"
```

### 7. Start the standby and verify standby mode

```bash
$ sudo systemctl start postgresql-13
$ psql
psql (13.4)
Type "help" for help.

postgres=# select pg_is_in_recovery();
 pg_is_in_recovery
--------------------
 t
(1 row)
```

```bash
postgres=# SELECT * FROM pg_stat_wal_receiver;
```

```text
-[ RECORD 1 ]---------+-------------------------------------------------------------------------------------------
pid                   | 659
status                | streaming
receive_start_lsn     | 0/3000000
receive_start_tli     | 1
written_lsn           | 0/3000348
flushed_lsn           | 0/3000348
received_tli          | 1
last_msg_send_time    | 2021-08-18 13:46:27.403973+00
last_msg_receipt_time | 2021-08-18 13:46:27.404131+00
latest_end_lsn        | 0/3000348
latest_end_time       | 2021-08-18 13:45:57.315332+00
slot_name             |
sender_host           | pg0
sender_port           | 5432
conninfo              | user=repmgr passfile=/home/postgres/.pgpass channel_binding=prefer connect_timeout=2 dbname=replication host=pg0 port=5432 application_name=pg1 fallback_application_name=walreceiver sslmode=prefer sslcompression=0 ssl_min_protocol_version=TLSv1.2 gssencmode=prefer krbsrvname=postgres target_session_attrs=any
```

**Verify replication from the primary side:**

```bash
postgres=# \x
Expanded display is on.
postgres=# select * from pg_stat_replication;
```

```text
-[ RECORD 1 ]----+------------------------------
pid              | 3091
usesysid         | 16384
usename          | repmgr
application_name | pg1
client_addr      | 10.206.6.63
client_hostname  |
client_port      | 39598
backend_start    | 2021-08-18 13:42:26.396195+00
backend_xmin     |
state            | streaming
sent_lsn         | 0/3000260
write_lsn        | 0/3000260
flush_lsn        | 0/3000260
replay_lsn       | 0/3000260
write_lag        |
flush_lag        |
replay_lag       |
sync_priority    | 0
sync_state       | async
reply_time       | 2021-08-18 13:44:46.766937+00
```

### 8. Register the standby

Run the following from the standby side:

```bash
$ /usr/pgsql-13/bin/repmgr -f /etc/repmgr/13/repmgr.conf standby register
INFO: connecting to local node "pg1" (ID: 2)
INFO: connecting to primary database
WARNING: --upstream-node-id not supplied, assuming upstream node is primary (node ID 1)
INFO: standby registration complete
NOTICE: standby node "pg1" (ID: 2) successfully registered
```

Verify the standby registration from the primary side:

```bash
$ /usr/pgsql-13/bin/repmgr -f /etc/repmgr/13/repmgr.conf cluster show
ID | Name | Role    | Status    | Upstream | Location | Priority | Timeline | Connection string
---+------+---------+-----------+----------+----------+----------+----------+------------------------------------------------------
1  | pg0  | primary | * running |          | default  | 100      | 1        | host=pg0 user=repmgr dbname=repmgr connect_timeout=2
2  | pg1  | standby |   running | pg0      | default  | 100      | 1        | host=pg1 user=repmgr dbname=repmgr connect_timeout=2
```

> The same information is available from the standby side as well.

### 9. Start the repmgrd daemon

The `repmgrd` daemon monitors the cluster and automates failover. Start it via `systemd` on each node in the cluster:

```bash
sudo systemctl start repmgr13
```

Verify the current service status:

```bash
$ repmgr -f /etc/repmgr/13/repmgr.conf service status
 ID | Name | Role    | Status    | Upstream | repmgrd | PID  | Paused? | Upstream last seen
----+------+---------+-----------+----------+---------+------+---------+--------------------
 1  | pg0  | primary | * running |          | running | 3923 | no      | n/a
 2  | pg1  | standby |   running | pg0      | running | 899  | no      | 0 second(s) ago
```

> **Note:** The daemon service often fails to start by default because the directory for the PID file is missing. If this happens, create it manually:
> ```bash
> sudo mkdir -p /run/repmgr
> sudo systemctl start repmgr13
> ```

### 10. Add a new node (third node) to the cluster

To add more nodes, repeat the same procedure described above on node 3 (`pg2`), starting from the **Clone the standby from primary** step — cloning, registering, starting PostgreSQL, and starting the repmgr daemon service.

Once complete, verify the cluster topology:

```bash
$ /usr/pgsql-13/bin/repmgr -f /etc/repmgr/13/repmgr.conf cluster show
ID | Name | Role    | Status    | Upstream | Location | Priority | Timeline | Connection string
---+------+---------+-----------+----------+----------+----------+----------+------------------------------------------------------
1  | pg0  | primary | * running |          | default  | 100      | 1        | host=pg0 user=repmgr dbname=repmgr connect_timeout=2
2  | pg1  | standby |   running | pg0      | default  | 100      | 1        | host=pg1 user=repmgr dbname=repmgr connect_timeout=2
3  | pg2  | standby |   running | pg0      | default  | 100      | 1        | host=pg2 user=repmgr dbname=repmgr connect_timeout=2
```

```bash
$ repmgr -f /etc/repmgr/13/repmgr.conf service status
 ID | Name | Role    | Status    | Upstream | repmgrd | PID  | Paused? | Upstream last seen
----+------+---------+-----------+----------+---------+------+---------+--------------------
 1  | pg0  | primary | * running |          | running | 3923 | no      | n/a
 2  | pg1  | standby |   running | pg0      | running | 899  | no      | 1 second(s) ago
 3  | pg2  | standby |   running | pg0      | running | 3074 | no      | 1 second(s) ago
```

## Tested Versions

| Component | Versions |
|-----------|----------|
| PostgreSQL | 11, 12, 13, 14, 15 |
| repmgr | 4.2 or above |
