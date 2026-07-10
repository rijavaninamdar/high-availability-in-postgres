# Patroni: Configuration and Bootstrapping

*Author: Rijavan Inamdar*

---

## Introduction

Patroni is an HA framework/template for PostgreSQL. This article details how to set up a Patroni cluster from the ground up, through to bootstrapping the new cluster. The overall setup can be divided into the following stages.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Configuration and Bootstrapping](#configuration-and-bootstrapping)
- [Starting the Patroni Cluster](#starting-the-patroni-cluster)
- [Second and Third Nodes Joining the Cluster](#second-and-third-nodes-joining-the-cluster)
- [Tested Versions](#tested-versions)

---

## Prerequisites

### 1. Understanding the Patroni Architecture

If you're not already familiar with Patroni's architecture, start with the [Patroni - Architecture and Overview](#) article.

### 2. Setting Up an etcd Cluster

Patroni relies on an underlying Distributed Configuration Store (DCS). We recommend **etcd** as the DCS for Patroni — see [Setting up etcd for Patroni](#) for details. This is the stage from which installation and setup begins.

### 3. Configuring Linux Watchdog

Fencing a cluster against split-brain situations is important. While this is considered optional for test setups, it's **mandatory and highly recommended** for production and other critical use cases. Configure Linux Watchdog on every node of the cluster.

### 4. Installing Patroni Software

The Patroni software installed on each node is responsible for managing the PostgreSQL instance on that node — starting, stopping, promoting, demoting, and resyncing it with other nodes. Patroni communicates with the DCS (etcd) to understand the current topology, configuration, and leader elections. See [Installation of Patroni Software](#) for details.

> **Note:** Every node in the Patroni cluster needs PostgreSQL binaries installed. Refer to your distribution's PostgreSQL installation guide for detailed steps.

---

## Configuration and Bootstrapping

Once all prerequisite steps above are complete, the cluster is ready for final configuration and bootstrapping.

### Step 1 — Create the Configuration Directory

```bash
sudo mkdir /etc/patroni
sudo chown postgres:postgres /etc/patroni
```

### Step 2 — Create the Patroni Configuration File

Create a YAML configuration file for Patroni (e.g., `pg0.yml`) under `/etc/patroni`:

```yaml
scope: stampede
name: pg0

restapi:
  listen: 0.0.0.0:8008
  connect_address: pg0:8008

etcd:
  host: pg0:2379

bootstrap:
  # this section will be written into Etcd:///config after initializing new cluster
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
#    master_start_timeout: 300
#    synchronous_mode: false
    postgresql:
      use_pg_rewind: true
      use_slots: true
      parameters:
        wal_level: replica
        hot_standby: "on"
        logging_collector: 'on'
        max_wal_senders: 5
        max_replication_slots: 5
        wal_log_hints: "on"
        archive_mode: "on"
        archive_timeout: 600s
        archive_command: "cp -f %p /home/postgres/archived/%f"
      recovery_conf:
        restore_command: cp /home/postgres/archived/%f %p

  # some desired options for 'initdb'
  initdb:  # Note: It needs to be a list (some options need values, others are switches)
  - encoding: UTF8
  - data-checksums

  pg_hba:  # Add following lines to pg_hba.conf after running 'initdb'
  - host replication replicator 192.168.50.1/24 md5
  - host replication replicator 127.0.0.1/32 trust
  - host all all 192.168.50.1/24 md5
  - host all all 0.0.0.0/0 md5
#  - hostssl all all 0.0.0.0/0 md5

  # Additional script to be launched after initial cluster creation (will be passed the connection URL as parameter)
  # post_init: /usr/local/bin/setup_cluster.sh

  # Some additional users which need to be created after initializing new cluster
  users:
    admin:
      password: admin
      options:
        - createrole
        - createdb

postgresql:
  listen: 0.0.0.0:5432
  connect_address: pg0:5432
  data_dir: "/var/lib/pgsql/12/data"
  bin_dir: "/usr/pgsql-12/bin"
  # config_dir:
  pgpass: /tmp/pgpass0
  authentication:
    replication:
      username: replicator
      password: vagrant
    superuser:
      username: postgres
      password: vagrant
  parameters:
    unix_socket_directories: '/var/run/postgresql'

watchdog:
  mode: required # Allowed values: off, automatic, required
  device: /dev/watchdog
  safety_margin: 5

tags:
    nofailover: false
    noloadbalance: false
    clonefrom: false
    nosync: false
```

> ⚠️ **Important:** Because indentation in YAML files carries structural meaning, downloading a sample YAML file is strongly preferred over copy-pasting the content above — indentation issues can cause hard-to-troubleshoot problems. Alternatively, download the file directly:
>
> ```bash
> cd /etc/patroni
> curl -LO https://raw.githubusercontent.com/jobinau/pgscripts/main/patroni/patroni.yml
> ```
>
> Edit the sample file to match your environment. If manual editing is required, use **two spaces** for indentation and avoid tab characters.

### Configuration Notes

- **`scope: stampede`** defines the cluster name — in this example, `stampede` is the name of the multi-node cluster, and this particular node is named `pg0`.
- **`bootstrap`** contains entries written to the DCS (etcd in this case). Its `postgresql` subsection holds cluster-wide PostgreSQL parameters that are enforced across all instances at startup — per the configuration hierarchy, these values live in the DCS.
- **`postgresql`** (top-level section) defines a few additional PostgreSQL-related parameters — including the location of the PostgreSQL binaries and data directory on this node. Per the hierarchy, these settings are **not** part of the DCS.
- **`pg_hba`** entries are used when initializing the PostgreSQL instance during bootstrapping, and are inserted into `pg_hba.conf` under the data directory. Make sure these entries allow the cluster's PostgreSQL instances to communicate with each other.
- **`watchdog`** is optional, but highly recommended for production clusters. This section defines the watchdog interface and safety margin.
- The `replicator` and `postgres` PostgreSQL user passwords are both set to `vagrant` in the sample — be sure to change these for your environment.

### WAL Archiving Setup

As general practice, it's usually better to store WAL archives on a separate storage system, made available to all nodes via a network-shared mechanism such as NFS:

```bash
sudo su - postgres -c "mkdir /home/postgres/archived"
```

> We recommend using **pgBackRest** for WAL archiving and database backups — see [pgBackRest with Patroni](#) for details.

If using a local mount point for WAL archiving, that location must be readable and writable by the `postgres` user:

```bash
sudo chown postgres:postgres /home/postgres/archived
```

The Patroni configuration file must also be readable by the `postgres` user, since the Patroni service runs under that account:

```bash
sudo chown postgres:postgres /etc/patroni/pg0.yml
```

---

## Starting the Patroni Cluster

First, disable the systemd service for PostgreSQL, to prevent it from starting independently of Patroni:

```bash
sudo systemctl disable postgresql-11
```

> The PostgreSQL instance in a Patroni cluster must be brought up and managed **by Patroni**, not by systemd directly.

As a best practice, run the Patroni service under the same `postgres` OS user account used to run PostgreSQL itself.

### Start the First Node

Confirm you're running as the `postgres` user:

```bash
$ whoami
postgres
```

Start Patroni, specifying the configuration file:

```bash
patroni /etc/patroni/pg0.yml &> patroni.log &
```

This bootstraps the Patroni cluster. The first node becomes the master by default, and Patroni ensures a master is always available going forward. Example output:

```
2018-09-10 10:49:32,945 INFO: Selected new etcd server http://192.168.50.xx:2379
2018-09-10 10:49:32,964 INFO: Lock owner: None; I am pg0
2018-09-10 10:49:32,971 INFO: trying to bootstrap a new cluster
The files belonging to this database system will be owned by user "postgres".
This user must also own the server process.

The database cluster will be initialized with locale "en_US.UTF-8".
The default text search configuration will be set to "english".

Data page checksums are enabled.

fixing permissions on existing directory /var/lib/pgsql/10/data ... ok
creating subdirectories ... ok
selecting default max_connections ... 100
selecting default shared_buffers ... 128MB
selecting dynamic shared memory implementation ... posix
creating configuration files ... ok
running bootstrap script ... ok
performing post-bootstrap initialization ... ok
syncing data to disk ... ok

WARNING: enabling "trust" authentication for local connections
You can change this by editing pg_hba.conf or using the option -A, or
--auth-local and --auth-host, the next time you run initdb.

Success. You can now start the database server using:

    /usr/pgsql-10/bin/pg_ctl -D /var/lib/pgsql/10/data -l logfile start

2018-09-10 10:49:38,071 INFO: postmaster pid=1499
2018-09-10 10:49:38.072 UTC [1499] LOG:  listening on IPv4 address "0.0.0.0", port 5432
2018-09-10 10:49:38.077 UTC [1499] LOG:  listening on Unix socket "./.s.PGSQL.5432"
2018-09-10 10:49:38.101 UTC [1499] LOG:  redirecting log output to logging collector process
2018-09-10 10:49:38.101 UTC [1499] HINT:  Future log output will appear in directory "log".
localhost:5432 - rejecting connections
localhost:5432 - accepting connections
2018-09-10 10:49:38,189 INFO: establishing a new patroni connection to the postgres cluster
2018-09-10 10:49:38,203 INFO: running post_bootstrap
2018-09-10 10:49:38,218 INFO: Software Watchdog activated with 25 second timeout, timing slack 15 seconds
2018-09-10 10:49:38,229 INFO: initialized a new cluster
2018-09-10 10:49:48,216 INFO: Lock owner: pg0; I am pg0
2018-09-10 10:49:48,232 INFO: Lock owner: pg0; I am pg0
2018-09-10 10:49:48,249 INFO: no action.  i am the leader with the lock
2018-09-10 10:49:58,224 INFO: Lock owner: pg0; I am pg0
```

Verify the status of the single-node "cluster":

```bash
patronictl -c /etc/patroni/pg0.yml list stampede
```

---

## Second and Third Nodes Joining the Cluster

### Node pg1

Copy the configuration file to node `pg1`, renaming it in the process:

```bash
scp /etc/patroni/pg0.yml postgres@pg1:/etc/patroni/pg1.yml
```

On node `pg1`, adjust ownership and finalize the file:

```bash
sudo chown postgres:postgres /etc/patroni/pg0.yml
sudo mv /etc/patroni/pg0.yml /etc/patroni/pg1.yml
sudo vi /etc/patroni/pg1.yml
```

Find and replace `pg0` with `pg1` (in `vi`: `%s/pg0/pg1`), then start Patroni:

```bash
patroni /etc/patroni/pg1.yml &> patroni.log &
patronictl -c /etc/patroni/pg1.yml list stampede
```

Node `pg1` will join the cluster, with its PostgreSQL instance coming up as a replica. Patroni initiates a backup from the current primary — using the configured backup/restore method (`pg_basebackup` by default) — to provision the new node and start it as a standby.

> **Note:** There's no need to manually back up the master to set up replicas — Patroni handles this automatically in the background, and can also integrate other backup tools besides `pg_basebackup`.

### Node pg2

Repeat the same procedure for node `pg2`:

```bash
scp /etc/patroni/pg0.yml postgres@pg2:/etc/patroni/pg2.yml
```

```bash
sudo chown postgres:postgres /etc/patroni/pg0.yml
sudo mv /etc/patroni/pg0.yml /etc/patroni/pg2.yml
sudo vi /etc/patroni/pg2.yml
```

Replace `pg0` with `pg2`, then start Patroni:

```bash
patroni /etc/patroni/pg2.yml &> patroni.log &
patronictl -c /etc/patroni/pg2.yml list stampede
```

Node `pg2` will join the cluster, with its PostgreSQL instance becoming another replica.

> ⚠️ **Important:** This procedure doesn't cover restarting the Patroni service after a reboot. For that, refer to Patroni's official repository — specifically, the `systemd` unit file for `patroni.service`.

---

## Tested Versions

This article has been tested against the following versions:

| Component | Version |
|---|---|
| PostgreSQL | 11, 12, 13, 14, 15, 16 |
| Patroni | 1.5 or above |
