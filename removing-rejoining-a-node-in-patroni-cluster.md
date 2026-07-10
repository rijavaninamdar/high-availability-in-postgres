# Patroni: How to Take Down, Rejoin, or Remove a Node from a Cluster

## Introduction

Different situations can lead to the decision to remove a node from a Patroni cluster — a failed node that needs replacing, or even hardware resource downsizing. This article walks through the right set of steps to remove a node from a Patroni cluster, or to rejoin a node to a cluster after repair or maintenance.

Removing a node might involve the following steps:

1. Stopping the Patroni service and PostgreSQL on the node, for temporary removal
2. Shutting down the host node, if it's a permanent removal
3. Removing the node from the etcd registry

## Process

### 1. Stop the Patroni service

Since Patroni is responsible for bringing up PostgreSQL on a node, stopping Patroni on that node will also stop PostgreSQL. If the Patroni service is managed by `systemd`, stopping it is as simple as:

```bash
$ sudo systemctl stop patroni
```

A status check confirms the service is no longer running:

```bash
$ sudo systemctl status patroni
● patroni.service - Runners to orchestrate a high-availability PostgreSQL
   Loaded: loaded (/etc/systemd/system/patroni.service; enabled; vendor preset: disabled)
   Active: inactive (dead) since Tue 2020-01-14 05:12:27 UTC; 7s ago
  Process: 12483 ExecStart=/usr/bin/patroni /etc/patroni/patroni.yml (code=exited, status=0/SUCCESS)
 Main PID: 12483 (code=exited, status=0/SUCCESS)
```

It's also good practice to verify that the PostgreSQL process is no longer running:

```bash
$ ps -eaf | grep postgres
```

Within a few seconds, Patroni will remove the node from its list. For example, the node list changes from:

```bash
-bash-4.2$ patronictl -c /etc/patroni/patroni.yml list
+----------+--------+------+--------+---------+----+-----------+
| Cluster  | Member | Host |  Role  |  State  | TL | Lag in MB |
+----------+--------+------+--------+---------+----+-----------+
| stampede |  pg0   | pg0  | Leader | running |  2 |           |
| stampede |  pg1   | pg1  |        | running |  2 |         0 |
| stampede |  pg2   | pg2  |        | running |  2 |         0 |
+----------+--------+------+--------+---------+----+-----------+
```

to:

```bash
$ patronictl -c /etc/patroni/patroni.yml list
+----------+--------+------+--------+---------+----+-----------+
| Cluster  | Member | Host |  Role  |  State  | TL | Lag in MB |
+----------+--------+------+--------+---------+----+-----------+
| stampede |  pg1   | pg1  |        | running |  3 |         0 |
| stampede |  pg2   | pg2  | Leader | running |  3 |           |
+----------+--------+------+--------+---------+----+-----------+
```

As shown, if the current Leader/Primary is removed, a standby is automatically promoted to take over the Leader role.

> **Note:** It may take a few seconds for the node to disappear from the Patroni node list. It may first show up with a `stopped` state for a short while, as seen below, before disappearing entirely:
>
> ```bash
> $ patronictl -c /etc/patroni/patroni.yml list
> +----------+--------+------+--------+---------+----+-----------+
> | Cluster  | Member | Host |  Role  |  State  | TL | Lag in MB |
> +----------+--------+------+--------+---------+----+-----------+
> | stampede |  pg0   | pg0  |        | stopped |    |   unknown |
> ```

### 2. Rejoin the cluster

At this stage, the node/server is available for repair, patching, etc., and can potentially rejoin the cluster.

Simply starting the Patroni service on the node brings it back into the cluster:

```bash
$ sudo systemctl start patroni
$ patronictl -c /etc/patroni/patroni.yml list
+----------+--------+------+--------+---------+----+-----------+
| Cluster  | Member | Host |  Role  |  State  | TL | Lag in MB |
+----------+--------+------+--------+---------+----+-----------+
| stampede |  pg0   | pg0  |        | running |  3 |         0 |
| stampede |  pg1   | pg1  |        | running |  3 |         0 |
| stampede |  pg2   | pg2  | Leader | running |  3 |           |
+----------+--------+------+--------+---------+----+-----------+
```

### 3. Permanently remove a node

The steps for permanently removing a node are similar to those for temporary removal above, but additionally involve shutting down the server host and disconnecting/removing the node from the distributed consensus store (etcd).

Once Patroni has been stopped on the node as described earlier, shut down the node:

```bash
$ sudo init 0
```

or

```bash
$ sudo systemctl halt
```

### 4. Remove the node from etcd

Removing/unregistering a node from the etcd cluster can be done from any of the remaining cluster nodes. Connect to any remaining node and confirm the node to be removed is still part of the member list:

```bash
$ etcdctl member list
22bf510ff95e289: name=pg2 peerURLs=http://192.168.50.30:2380 clientURLs=http://192.168.50.30:2379 isLeader=false
3b66e040d5be4514: name=pg0 peerURLs=http://192.168.50.10:2380 clientURLs=http://192.168.50.10:2379 isLeader=false
f72964da710cd1d9: name=pg1 peerURLs=http://192.168.50.20:2380 clientURLs=http://192.168.50.20:2379 isLeader=true
```

Note the ID of the member to be removed, and use it in the `etcdctl` command:

```bash
$ etcdctl member remove 3b66e040d5be4514
Removed member 3b66e040d5be4514 from cluster
```

Verify the member has been removed:

```bash
$ etcdctl member list
22bf510ff95e289: name=pg2 peerURLs=http://192.168.50.30:2380 clientURLs=http://192.168.50.30:2379 isLeader=false
f72964da710cd1d9: name=pg1 peerURLs=http://192.168.50.20:2380 clientURLs=http://192.168.50.20:2379 isLeader=true
```

> **Important:** Also remove the node's host/IP reference from the Patroni configuration file, under the `etcd` section.

## Tested Versions

| Component | Versions |
|-----------|----------|
| PostgreSQL | 11, 12, 13, 14, 15, 16 |
| Patroni | 1.5 or above |
