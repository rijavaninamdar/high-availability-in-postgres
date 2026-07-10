# Backing Up a Patroni Cluster

## Introduction

One compelling reason to maintain a backup routine for etcd and Patroni is that configuration changes made with `patronictl edit-config` are written directly to the etcd database. These changes propagate to PostgreSQL's custom configuration files under the data directory, but they **do not** update the YAML configuration files used to bootstrap Patroni.

This means that any configuration changes applied via `patronictl edit-config` will **not** be reapplied if a new Patroni cluster is bootstrapped from scratch.

Since `patronictl edit-config` is the recommended, safe way to make configuration changes in a Patroni cluster, backing up the etcd database is essential to preserving those changes.

## Process

### 1. Back up the etcd database

Use `etcdctl` to take a copy of the etcd database:

```bash
etcdctl backup --data-dir /var/lib/etcd/postgresql --backup-dir /root/etcd_backup
```

> **Important:** If etcd's WAL files are stored in a separate directory, use the `--wal-dir` and `--backup-wal-dir` options to specify the source and destination locations respectively.

> **Note:** Backing up the etcd database isn't always strictly necessary, but it's a prudent precaution to take.

### 2. Back up the main configuration files

Keep backups of the following core configuration files:

| Component | Typical Path |
|-----------|-------------|
| etcd      | `/etc/etcd/etcd.conf` |
| Patroni   | `/etc/patroni/postgresql.yml` |

> **Note:** These paths can vary between environments — this is especially common for Patroni's `postgresql.yml` file.

### 3. Back up `patroni.dynamic.json`

Take a backup of the `patroni.dynamic.json` file located in your PostgreSQL data directory, and store it alongside your other Patroni-related backups.

Typical path:

```
/var/lib/pgsql/14/data/patroni.dynamic.json
```

> **Important:** `patroni.dynamic.json` is the source file from which Patroni settings stored in etcd are typically restored.

> **Note:** Substitute your PostgreSQL major version for `14` in the example path above if you're running a different version.

## Considerations

While etcd is designed to be resilient and can automatically recover from a number of temporary cluster failures, it's possible for a cluster to permanently lose enough members to irrevocably lose quorum.

Having proper backups in place allows you to recreate the cluster **without losing any PostgreSQL configuration changes** made via `patronictl edit-config` while Patroni was running.
