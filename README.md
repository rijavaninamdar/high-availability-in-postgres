# High Availability in PostgreSQL

A collection of practical, hands-on guides covering PostgreSQL High Availability (HA) using **Patroni**, **etcd**, **HAProxy**, streaming replication, and the **EDB HA stack** (EFM, PEM, Barman).

These notes are written from real lab/production experience and are meant as a quick reference for setting up, operating, and troubleshooting HA PostgreSQL clusters.

## 📖 Table of Contents

### 🏗️ Architecture & Concepts
- [Patroni Architecture Overview](./patroni-architecture-overview.md) — how Patroni, etcd, and PostgreSQL work together to provide automated failover.

### ⚙️ Installation & Configuration
- [Patroni Installation](./patroni-installation.md) — installing Patroni and its dependencies.
- [Patroni Installation on Ubuntu](./patroni-installation-ubuntu.md) — Ubuntu-specific installation steps.
- [Setting Up etcd](./patroni-setting-up-etcd.md) — deploying and configuring an etcd cluster for Patroni.
- [Setting Up etcd 3.5 and Above](./patroni-setting-up-etcd-3.5-and-above.md) — changes and considerations for newer etcd versions.
- [Patroni Configuration & Bootstrapping](./patroni-configuration-bootstrapping.md) — bootstrapping a new Patroni cluster and key configuration options.

### 🔁 Replication
- [PostgreSQL Streaming (Physical) Replication](./postgresql-streaming-physical-replication.md) — setting up native streaming replication.
- [PostgreSQL Cascade Replication](./postgresql-cascade-replication.md) — chaining replicas to reduce load on the primary.
- [PostgreSQL Synchronous vs Asynchronous Replication](./postgresql-synchronous-asynchronous-replication.md) — trade-offs and configuration for each mode.

### 🔧 Cluster Operations
- [Adding a New Node to a Patroni Cluster](./adding-a-new-node-to-a-patroni-cluster.md) — safely scaling out an existing cluster.
- [Patroni Switchover Operation](./patroni-switchover-operation.md) — performing planned leader changes without downtime.
- [Backing Up a Patroni Cluster](./backing-up-a-patroni-cluster.md) — preserving etcd and configuration state for disaster recovery.

### 🌐 Connection Routing
- [Patroni + HAProxy Connection Routing](./patroni-haproxy-connection-routing.md) — routing client traffic to the correct node using HAProxy.

### 🧩 EDB HA Stack
- [EDB EPAS HA Lab: EFM, PEM, Barman](./edb-epas-ha-lab-efm-pem-barman.md) — building an HA lab using EDB Failover Manager, Postgres Enterprise Manager, and Barman.

## 🎯 Who This Is For

Database administrators, SREs, and engineers who are:
- Setting up a new HA PostgreSQL environment
- Migrating from a manual failover process to Patroni
- Looking for quick, tested reference commands during cluster operations

## 🛠️ Tech Covered

`PostgreSQL` · `Patroni` · `etcd` · `HAProxy` · `EDB Postgres Advanced Server` · `EFM` · `PEM` · `Barman`

## 📌 Notes

- Examples are based on RHEL/Ubuntu environments and PostgreSQL versions 14–16; adjust paths and package names for your specific OS and PostgreSQL version.
- These are living documents — commands, paths, and package repositories may change over time, so always cross-check against current official documentation before applying changes to production systems.

## 🤝 Contributing

Suggestions, corrections, and pull requests are welcome. If you spot an outdated command or a broken step, please let me know

## 📄 License

Feel free to use, share, and adapt this content for learning purposes.
