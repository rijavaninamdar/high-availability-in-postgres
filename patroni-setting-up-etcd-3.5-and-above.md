# Patroni: How to Set Up etcd 3.5 and Above

*Author: Rijavan Inamdar*

---

## Introduction

etcd 3.5 and above switched to **API version 3** as the default. Along with this change, our recommendations and setup guidance have been updated based on lessons learned across multiple installations and customer environments.

Some of the notable changes in etcd, and in our recommended configuration approach, include:

- API version change to **v3**
- Configuration expressed in **YAML** (instead of environment variables)
- New bootstrapping options
- A corresponding change in Patroni's configuration (the `etcd:` section becomes `etcd3:` — see [Patroni's etcdv3 documentation](https://patroni.readthedocs.io/en/latest/yaml_configuration.html#etcdv3) for details)

## Table of Contents

- [Step 1: Install etcd](#step-1-install-etcd)
- [Step 2: Prepare the Configuration File](#step-2-prepare-the-configuration-file)
- [Step 3: Start the etcd Service](#step-3-start-the-etcd-service)
- [Step 4: Verify the etcd Cluster](#step-4-verify-the-etcd-cluster)
- [Tested Versions](#tested-versions)

---

## Step 1: Install etcd

Install the etcd packages or binaries on each node. We recommend installing packages from the [Percona repository](https://docs.percona.com/percona-software-repositories/installing.html).

**Advantages of Percona packages:**

- QA-tested packages maintained by Percona
- A sample etcd configuration provided under `/etc/etcd`
- A ready-made `systemd` unit configuration for the etcd service

The exact installation command depends on your host OS's package manager. For example, on RHEL 8:

```bash
sudo dnf install etcd
```

---

## Step 2: Prepare the Configuration File

> **Note:** It's possible to bootstrap the etcd cluster as a single-node cluster first, then add nodes one at a time — this is the approach we continue to recommend for novice users, [per the etcd documentation](https://etcd.io/docs/latest/op-guide/clustering/). This article instead illustrates bootstrapping the etcd cluster with **all nodes at once**.

Prepare a YAML configuration file for each node, and place it at `/etc/etcd/etcd.conf.yaml`:

```bash
$ cat /etc/etcd/etcd.conf.yaml
```

```yaml
name: 'pg1'
data-dir: /var/lib/etcd
initial-advertise-peer-urls: http://10.3.188.140:2380
listen-peer-urls: http://10.3.188.140:2380
advertise-client-urls: http://10.3.188.140:2379
listen-client-urls: http://10.3.188.140:2379,http://localhost:2379
initial-cluster: pg1=http://10.3.188.140:2380,pg2=http://10.3.188.138:2380,pg3=http://10.3.188.104:2380
initial-cluster-state: new
initial-cluster-token: pgcluster
```

**Key fields:**

| Field | Description |
|---|---|
| `name` | The node's name |
| `data-dir` | Where etcd stores its data — this location must be writable by the `etcd` user |
| `listen-client-urls` | Where the etcd server listens for client connections (e.g., from `etcdctl`) |
| `initial-cluster` | Lists every node in the cluster, along with the port each etcd instance runs on |

Repeat this step for each node, adjusting `name`, the IP addresses, and URLs accordingly.

---

## Step 3: Start the etcd Service

```bash
sudo systemctl start etcd
```

> ⚠️ **Important:** Start the etcd service on **all nodes** with as little delay as possible between them. Opening a separate terminal to each node and running the command in close succession helps minimize this delay.

---

## Step 4: Verify the etcd Cluster

```bash
etcdctl member list
```

**Example output:**

```
8286431a18928d1f, started, pg2, http://10.3.188.138:2380, http://10.3.188.138:2379, false
e054f84d75eb4097, started, pg3, http://10.3.188.104:2380, http://10.3.188.104:2379, false
fcba16da26d50257, started, pg1, http://10.3.188.140:2380, http://10.3.188.140:2379, false
```

Confirm all nodes appear in the member list and that the cluster has successfully reached quorum.

---

## Tested Versions

This article has been tested against the following versions:

| Component | Version |
|---|---|
| PostgreSQL | 12, 13, 14, 15, 16, 17 |
| etcd | 3.5 |

---

## Further Reading

- [Percona: etcd setup for Percona Distribution for PostgreSQL](https://docs.percona.com/postgresql/17/solutions/ha-etcd-config.html)
- [Percona blog: Upgrading to the New etcd Version From 3.3 for Patroni](https://www.percona.com/blog/upgrading-to-the-new-etcd-version-from-3-3-for-patroni/)
- [Patroni documentation: etcdv3 configuration](https://patroni.readthedocs.io/en/latest/yaml_configuration.html#etcdv3)
- [etcd clustering guide](https://etcd.io/docs/latest/op-guide/clustering/)
