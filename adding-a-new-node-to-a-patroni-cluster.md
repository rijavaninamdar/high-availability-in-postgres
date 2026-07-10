# Adding a New Node to a Patroni Cluster

## Introduction

It's a very common requirement to add new nodes to a Patroni cluster — often as a step to replace an old machine or problematic hardware.

The new node can use a different Linux distribution/version (see the note at the end of this guide) and can pull packages from different repositories. If the node needs to run etcd, the etcd version can be newer, but it **must** use the same protocol version as the existing etcd cluster.

## Process

For illustration purposes, this guide refers to an existing cluster with the following nodes:

```bash
[postgres@node0 ~]$ patronictl list
+ Cluster: stampede (7486346636944005175) -+-----------+
| Member | Host | Role    | State     | TL | Lag in MB |
+--------+------+---------+-----------+----+-----------+
| pg1    | pg1  | Leader  | running   |  7 |           |
| pg2    | pg2  | Replica | streaming |  7 |         0 |
| pg3    | pg3  | Replica | streaming |  7 |         0 |
+--------+------+---------+-----------+----+-----------+
```

All existing nodes run **RHEL 8** with PostgreSQL installed from the [PGDG repository](https://www.postgresql.org/download/linux/redhat/).

This guide adds a new node, **pg4**, which runs **RHEL 9** with PostgreSQL packages installed from the [Percona Repository](https://docs.percona.com/postgresql/16/installing.html).

### 1. Ensure connectivity

Add the new node `pg4` to the network and confirm it is reachable by name from all existing nodes, and vice versa:

```bash
[postgres@node0 ~]$ ping pg4
PING pg4 (192.168.96.5) 56(84) bytes of data.
64 bytes from pg4 (192.168.96.5): icmp_seq=1 ttl=64 time=0.097 ms
64 bytes from pg4 (192.168.96.5): icmp_seq=2 ttl=64 time=0.058 ms
64 bytes from pg4 (192.168.96.5): icmp_seq=3 ttl=64 time=0.047 ms
```

### 2. Install the necessary packages on the new node

Refer to the [Percona documentation for installing packages from the Percona repository](https://docs.percona.com/postgresql/16/installing.html).

Example for RHEL 9:

```bash
export TOOL=dnf
export PGVER=16

sudo $TOOL install -y epel-release
sudo $TOOL install -y http://repo.percona.com/yum/percona-release-latest.noarch.rpm
sudo percona-release setup ppg-$PGVER
sudo percona-release show

sudo dnf -qy module disable postgresql   # Disable the built-in module
sudo dnf install -y percona-postgresql$PGVER-server
sudo dnf install -y percona-pg_stat_monitor$PGVER
sudo dnf install -y percona-patroni
sudo dnf install -y etcd
sudo dnf install -y python3-etcd.noarch
```

### 3. Add a new etcd member *(optional)*

> This step is only required if the newly added node also needs to participate in the etcd cluster.

Run `etcdctl member add` from any existing etcd node:

```bash
etcdctl member add pg4 --peer-urls=http://192.168.96.5:2380
```

Expected output:

```text
Member 40213579e6b99ba5 added to cluster d71e0beb84ff332

ETCD_NAME="pg4"
ETCD_INITIAL_CLUSTER="pg2=http://192.168.96.3:2380,pg1=http://192.168.96.2:2380,pg4=http://192.168.96.5:2380,pg3=http://192.168.96.4:2380"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.96.5:2380"
ETCD_INITIAL_CLUSTER_STATE="existing"
```

This output must be used when configuring the newly added etcd member.

At this point, `etcd member list` will show the new node's status as `unstarted`:

```bash
$ etcdctl member list
12d494a29d9c1b5b, started, pg2, http://192.168.96.3:2380, http://192.168.96.3:2379, false
3f76cff908599ebc, started, pg1, http://192.168.96.2:2380, http://192.168.96.2:2379, false
40213579e6b99ba5, unstarted, , http://192.168.96.5:2380, , false
db51dc66eee467b9, started, pg3, http://192.168.96.4:2380, http://192.168.96.4:2379, false
```

Prepare the etcd configuration on the `pg4` machine using the information gathered above. Sample configuration in YAML format:

```yaml
# cat /etc/etcd/etcd.conf.yaml
name: 'pg4'
data-dir: /var/lib/etcd
initial-advertise-peer-urls: http://192.168.96.5:2380
listen-peer-urls: http://192.168.96.5:2380
advertise-client-urls: http://192.168.96.5:2379
listen-client-urls: http://192.168.96.5:2379,http://localhost:2379
initial-cluster: pg2=http://192.168.96.3:2380,pg1=http://192.168.96.2:2380,pg4=http://192.168.96.5:2380,pg3=http://192.168.96.4:2380
initial-cluster-state: existing
initial-cluster-token: pgcluster
auto-compaction-retention: '15m'
auto-compaction-mode: 'periodic'
snapshot-count: 100
max-wals: 2
auto-compaction-mode: revision
auto-compaction-retention: '100'
```

Start the etcd service and check the status:

```bash
$ sudo systemctl start etcd

$ etcdctl member list
12d494a29d9c1b5b, started, pg2, http://192.168.96.3:2380, http://192.168.96.3:2379, false
3f76cff908599ebc, started, pg1, http://192.168.96.2:2380, http://192.168.96.2:2379, false
40213579e6b99ba5, started, pg4, http://192.168.96.5:2380, http://192.168.96.5:2379, false
db51dc66eee467b9, started, pg3, http://192.168.96.4:2380, http://192.168.96.4:2379, false
```

This confirms the new node has been added to the etcd cluster.

### 4. Configure the new node as a Patroni node

Navigate to the Patroni configuration location and pull a sample/template configuration:

```bash
# cd /etc/patroni/
# curl -LO https://raw.githubusercontent.com/jobinau/pgscripts/refs/heads/main/patroni/patroni.yml
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  2381  100  2381    0     0   6417      0 --:--:-- --:--:-- --:--:--  6417
```

Make sure to update all parameters for the cluster and for this specific node.

### 5. Start the Patroni service to initialize the node

Once the Patroni service starts on the new node, Patroni will automatically initialize it and add it to the cluster.

Sample Patroni log output:

```text
2025-06-02 08:18:50,465 INFO: Selected new etcd server http://192.168.96.5:2379
2025-06-02 08:18:50,475 INFO: No PostgreSQL configuration items changed, nothing to reload.
2025-06-02 08:18:50,529 INFO: Lock owner: pg1; I am pg4
2025-06-02 08:18:50,592 INFO: trying to bootstrap from leader 'pg1'
2025-06-02 08:18:53,735 INFO: replica has been created using basebackup
2025-06-02 08:18:53,736 INFO: bootstrapped from leader 'pg1'
2025-06-02 08:18:54,047 INFO: postmaster pid=2859
2025-06-02 08:18:54.051 UTC [2859] LOG:  redirecting log output to logging collector process
2025-06-02 08:18:54.051 UTC [2859] HINT:  Future log output will appear in directory "log".
localhost:5432 - rejecting connections
localhost:5432 - rejecting connections
2025-06-02 08:18:54,921 INFO: Lock owner: pg1; I am pg4
2025-06-02 08:18:54,978 INFO: bootstrap from leader 'pg1' in progress
localhost:5432 - accepting connections
2025-06-02 08:18:55,105 INFO: Lock owner: pg1; I am pg4
2025-06-02 08:18:55,105 INFO: establishing a new patroni heartbeat connection to postgres
2025-06-02 08:18:55,238 INFO: no action. I am (pg4), a secondary, and following a leader (pg1)
2025-06-02 08:19:05,018 INFO: no action. I am (pg4), a secondary, and following a leader (pg1)
```

> **Important:** If the `glibc` version on the newly added node differs from the existing nodes, avoid using that node for any application connections. SQL statements might return incorrect results due to logical (collation-related) corruption.
