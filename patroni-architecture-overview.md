# Patroni: Architecture and Overview

*Author: Rijavan Inamdar*

---

## Introduction

PostgreSQL is known for its robustness. The process-based design of its architecture helps isolate user sessions from one another as separate OS processes. Even though issues affecting one session may affect the server as a whole, they seldom compromise its overall availability.

There are many High-Availability (HA) projects built around PostgreSQL, catering to different levels of availability requirements — such as **pgPool-II**, **repmgr**, and **PAF**. Many older-generation HA solutions for PostgreSQL are also documented on the [PostgreSQL wiki](https://wiki.postgresql.org/wiki/Replication,_Clustering,_and_Connection_Pooling). However, many modern applications demand higher availability than these traditional HA solutions can offer — and the availability standards set by cloud providers have pushed expectations even further.

Over several years, PostgreSQL has gained key features that a modern HA solution can leverage:

- Streaming replication
- Synchronous replication (both priority-based and quorum-based)
- Asynchronous replication
- Hot-standby
- The ability to re-sync a replica through `pg_rewind`

At the same time, open-source distributed consensus frameworks like **etcd** (by CoreOS) have gained momentum — the same technology leveraged by projects like Kubernetes.

## Table of Contents

- [What Is Patroni and What Does It Do?](#what-is-patroni-and-what-does-it-do)
- [Major Components](#major-components)
- [Key Benefits of Patroni](#key-benefits-of-patroni)
- [Caveats](#caveats)

---

## What Is Patroni and What Does It Do?

The [official documentation](https://patroni.readthedocs.io/) describes Patroni as a *"template for you to create your own customized, high-availability solution using Python and — for maximum accessibility — a distributed configuration store like ZooKeeper, etcd, Consul, or Kubernetes."*

In simpler terms, Patroni is an open-source project that provides a template for HA solutions, combining modern tools like etcd with PostgreSQL's built-in HA capabilities (as noted above). Patroni maintains the PostgreSQL HA replication cluster in a fully automated way, using the capabilities offered by its available components.

**Key behaviors:**

- Patroni ensures a master is always present in the cluster, and that master promotion happens without conflict or split-brain.
- It relies on a distributed consensus mechanism provided by infrastructure such as ZooKeeper, etcd, Consul, or Kubernetes.
- In a Patroni-managed replication cluster, the connection routing layer can automatically route connections to the master or to replicas.
- On master failure, application connections routed to it can be retried by the application, and Patroni ensures they're rerouted to the newly elected master. Read-only connections may not even notice if a standby replica is removed from the cluster.

### Failover and Recovery

In the event of master failure, Patroni holds an election to appoint a new master from among the eligible replicas. If the old master becomes available again, Patroni can automatically reintroduce it back into the cluster as a replica — using `pg_rewind` to resynchronize its data directory with the new master, rather than rebuilding it entirely from scratch. This saves both time and bandwidth.

### No Special Nodes

An interesting characteristic of Patroni is that it ensures there are no "special" nodes in the cluster: at election time, every node is a potential candidate to become the new master, unless explicitly marked with the `nofailover` tag. Patroni achieves this by keeping all necessary configuration in a centralized, distributed store.

> The Patroni project is maintained in a [public repository](https://github.com/patroni/patroni).

---

## Major Components

A typical Patroni setup is an open-source stack made up of the following components:

| Component | Role |
|---|---|
| **Connection Router** | Sits at the top of the stack (e.g., HAProxy). Talks to the Patroni instances to correctly identify the current master and replicas, routing client connections accordingly. |
| **Patroni Layer** | Manages the PostgreSQL instances running on each node — starting, stopping, promoting, and monitoring the health of each instance, among other responsibilities. |
| **Distributed Configuration System (DCS)** | An underlying consensus mechanism (e.g., etcd) that handles cluster management tasks like leader election, and acts as a distributed store for all configuration data. Notably, **all PostgreSQL parameter settings are stored in this layer.** |
| **Split-Brain Protection** *(optional)* | A Linux Watchdog mechanism — a hardware + OS combination that monitors whether a server is hanging or stalling. If Watchdog suspects a hang, it forces a server reboot to prevent split-brain. |

---

## Key Benefits of Patroni

- **Single-command switchover** — manual or automated master switchover with one command.
- **Continuous monitoring** — for service failure, with automatic restructuring of the replication topology for cluster healing.
- **Built-in re-sync automation** — automatically resynchronizes a failed node and brings it back into the cluster.
- **Customizable replica build process.**
- **API-driven, consensus-based configuration** — every configuration change requires consensus from the cluster, where nodes collectively decide whether to accept or reject it, making the process fool-proof.
- **Maintenance mode** — allows the entire cluster's automation to be paused.
- **Transparent connection routing infrastructure.**
- **Collective decision-making** — the distributed consensus algorithm ensures decisions (configuration changes, leader elections) are made collectively, preventing unilateral action and the mistakes that can come with it. This guarantees there is always a single source of truth for the entire cluster.
- **Optional Linux Watchdog integration** — protects against split-brain by ensuring an evicted master can't continue operating in isolation. Instead, it's rebooted and prepared to rejoin the cluster as a replica.

---

## Caveats

- Setting up a Patroni cluster is **complex** and requires real effort, since it relies on several external components.
- The Patroni agent directly controls PostgreSQL instances and their configuration, making it a somewhat **"invasive"** solution — all DBAs involved need to understand how it works and operates.
- The entry point relies on external mechanisms (e.g., HAProxy, or endpoints offered by cloud providers).
- The consensus mechanism is also external (etcd, ZooKeeper, or Consul) and may vary across different setups.
- These moving parts increase the difficulty of automated deployments, with added complexity depending on which components are chosen.
