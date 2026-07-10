# Patroni: Setting Up etcd for Patroni

*Author: Rijavan Inamdar*

---

## Introduction

Patroni can use any distributed key-value store with a consensus mechanism, such as ZooKeeper, etcd, or Consul. In Patroni terminology, this is called the **DCS** (Distributed Configuration Store).

This article focuses specifically on setting up **etcd** for Patroni, though other key-value stores work similarly. etcd provides powerful features and is the most widely used DCS option with Patroni — it's a stable solution that's proven itself over several years. **We recommend etcd for all Patroni installations.**

## Table of Contents

- [etcd in a Patroni Cluster](#etcd-in-a-patroni-cluster)
- [Sample Cluster Layout](#sample-cluster-layout)
- [Preparing the First Node](#preparing-the-first-node)
- [Adding a Second Node](#adding-a-second-node)
- [Adding a Third Node](#adding-a-third-node)
- [Final Verification](#final-verification)
- [Tested Versions](#tested-versions)

---

## etcd in a Patroni Cluster

Simply put, etcd is a key-value store — but a consensus must be reached across all cluster members before any key-value pair can be inserted or updated. This is achieved using the **Raft** consensus algorithm, making etcd a "key-value store with distributed consensus." It's part of the CoreOS project, and is used by many popular open-source projects, including Kubernetes. The [etcd GitHub repository](https://github.com/etcd-io/etcd) has more details.

**How leader election works:**

1. When a node comes up, the Patroni service on that node asks etcd to mark it as the **Leader** (Primary) node — in Patroni's vocabulary, the primary node is also referred to as the cluster's "Leader."
2. It's up to etcd to decide whether to grant that request. Before granting it, etcd uses distributed consensus to ensure only **one** node is ever assigned the leader key.
3. The node that receives the leader key becomes the leader. Patroni then brings up PostgreSQL as the primary instance on that node, while ensuring all other nodes become its standbys.

etcd serves as the underlying clustering infrastructure. A node doesn't need to run PostgreSQL or Patroni to participate in the etcd cluster — an etcd member can run on a dedicated machine, or alongside another service such as an application server or a dedicated HAProxy server.

---

## Sample Cluster Layout

This article uses a sample 3-node cluster:

| Node | Hostname | IP Address |
|---|---|---|
| Node 1 | `pg0` | `192.168.50.90` |
| Node 2 | `pg1` | `192.168.50.95` |
| Node 3 | `pg2` | `192.168.50.100` |

> ⚠️ **Important:** This article is written for **etcd 3.3 and older**. If you're installing a newer version (e.g., 3.5), refer to article **KB0049666** instead.

---

## Preparing the First Node

etcd is available in the standard repositories for most Linux distributions. On RHEL-like distributions:

```bash
sudo yum install etcd
```

etcd runs as a daemon and reads its configuration from `/etc`. Edit the configuration file to prepare it for use with Patroni:

```bash
sudo vi /etc/etcd/etcd.conf
```

**Configuration for the first node (`pg0`):**

```ini
ETCD_NAME=pg0
ETCD_INITIAL_CLUSTER="pg0=http://192.168.50.90:2380"
ETCD_INITIAL_CLUSTER_TOKEN="devops_token"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.50.90:2380"
ETCD_DATA_DIR="/var/lib/etcd/postgres.etcd"
ETCD_LISTEN_PEER_URLS="http://192.168.50.90:2380"
ETCD_LISTEN_CLIENT_URLS="http://192.168.50.90:2379,http://localhost:2379"
ETCD_ADVERTISE_CLIENT_URLS="http://192.168.50.90:2379"
```

Here, the etcd instance is named after its host: `ETCD_NAME=pg0`.

Start the etcd service, and enable it to start on boot:

```bash
sudo systemctl start etcd
sudo systemctl enable etcd
```

Since this is the first etcd node in the cluster, the service status will look like this:

```
$ sudo systemctl status etcd
etcd.service - Etcd Server
  Loaded: loaded (/usr/lib/systemd/system/etcd.service; disabled; vendor preset: disabled)
  Active: activating (start) since Thu 2017-09-21 08:46:52 UTC; 18s ago
Main PID: 2398 (etcd)
  CGroup: /system.slice/etcd.service
         └─2398 /usr/bin/etcd --name=pg0 --data-dir=/var/lib/etcd/postgres.etcd --listen-client-urls=http://192.168.50.90:2379,http://localhost:2379
```

> The `activating (start)` status can look confusing at first — it simply means etcd isn't yet a full cluster and is waiting for other nodes to join. Newer versions of etcd report this more clearly, showing `active (running)` instead.

---

## Adding a Second Node

Tell the first node that a second node is ready to join, using `etcdctl member add`:

```bash
etcdctl member add pg1 --peer-urls=http://192.168.50.95:2380
```

This adds the new member to etcd's internal key-value store and outputs the configuration needed for the new node:

```
Added member named pg1 with ID 9273ebfc32c6ae5 to cluster
ETCD_NAME="pg1"
ETCD_INITIAL_CLUSTER="pg1=http://192.168.50.95:2380,pg0=http://192.168.50.90:2380"
ETCD_INITIAL_CLUSTER_STATE="existing"
```

Install the etcd package on `pg1` (as done for `pg0` above), then edit its configuration file using the printed hints:

```bash
sudo vi /etc/etcd/etcd.conf
```

**Final configuration for `pg1`:**

```ini
ETCD_NAME="pg1"
ETCD_INITIAL_CLUSTER="pg1=http://192.168.50.95:2380,pg0=http://192.168.50.90:2380"
ETCD_INITIAL_CLUSTER_STATE="existing"
ETCD_INITIAL_CLUSTER_TOKEN="devops_token"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.50.95:2380"
ETCD_DATA_DIR="/var/lib/etcd/postgres.etcd"
ETCD_LISTEN_PEER_URLS="http://192.168.50.95:2380"
ETCD_LISTEN_CLIENT_URLS="http://192.168.50.95:2379,http://localhost:2379"
ETCD_ADVERTISE_CLIENT_URLS="http://192.168.50.95:2379"
```

Two notable differences from `pg0`'s config: `ETCD_INITIAL_CLUSTER` now lists **two** member instances, and `ETCD_INITIAL_CLUSTER_STATE` is set to `"existing"` rather than `"new"`.

Start the etcd service and enable it on boot:

```bash
sudo systemctl start etcd
sudo systemctl enable etcd
```

etcd is now a multi-node cluster. Verify membership with:

```bash
etcdctl member list
```

This should list both member instances.

---

## Adding a Third Node

Adding the third node follows the same process. You can run the `member add` command from `pg0` again, or from any node already in the cluster:

```bash
etcdctl member add pg2 --peer-urls=http://192.168.50.100:2380
```

This prints the configuration hints for the third node:

```
Added member named pg2 with ID 40ffb966c8d2a67b to cluster
ETCD_NAME="pg2"
ETCD_INITIAL_CLUSTER="pg1=http://192.168.50.95:2380,pg2=http://192.168.50.100:2380,pg0=http://192.168.50.90:2380"
ETCD_INITIAL_CLUSTER_STATE="existing"
```

Edit `/etc/etcd/etcd.conf` on `pg2`:

```bash
sudo vi /etc/etcd/etcd.conf
```

**Final configuration for `pg2`:**

```ini
ETCD_NAME="pg2"
ETCD_INITIAL_CLUSTER="pg2=http://192.168.50.100:2380,pg1=http://192.168.50.95:2380,pg0=http://192.168.50.90:2380"
ETCD_INITIAL_CLUSTER_STATE="existing"
ETCD_INITIAL_CLUSTER_TOKEN="devops_token"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.50.100:2380"
ETCD_DATA_DIR="/var/lib/etcd/postgres.etcd"
ETCD_LISTEN_PEER_URLS="http://192.168.50.100:2380"
ETCD_LISTEN_CLIENT_URLS="http://192.168.50.100:2379,http://localhost:2379"
ETCD_ADVERTISE_CLIENT_URLS="http://192.168.50.100:2379"
```

Start the etcd service on the third node, and enable autostart:

```bash
sudo systemctl start etcd
sudo systemctl enable etcd
```

---

## Final Verification

Verify the current etcd cluster using `member list`:

```bash
etcdctl member list
```

This should list all three members, similar to:

```
3b66e040d5be4514: name=pg0 peerURLs=http://192.168.50.90:2380 clientURLs=http://192.168.50.90:2379 isLeader=false
525ffefa211f535c: name=pg1 peerURLs=http://192.168.50.95:2380 clientURLs=http://192.168.50.95:2379 isLeader=true
5bb220e7274ca7f6: name=pg2 peerURLs=http://192.168.50.100:2380 clientURLs=http://192.168.50.100:2379 isLeader=false
```

Make sure all cluster nodes are listed, and that exactly one node is marked `isLeader=true`.

---

## Tested Versions

This article has been tested against the following versions:

| Component | Version |
|---|---|
| PostgreSQL | 12, 13, 14, 15, 16 |
| etcd | 3.4 |
