# Patroni: How to Perform a Switchover Operation

*Author: Rijavan Inamdar*

---

## Introduction

The primary server in a Patroni cluster can be gracefully switched from one cluster node to another. This is useful for performing node-specific maintenance activities. The database remains available throughout the process — the only caveat is that application connections may need to be re-established after the switchover completes.

## Table of Contents

- [Step 1: Initiate the Switchover](#step-1-initiate-the-switchover)
- [Step 2: Confirm the Current Primary](#step-2-confirm-the-current-primary)
- [Step 3: Choose a Candidate](#step-3-choose-a-candidate)
- [Step 4: Schedule the Switchover](#step-4-schedule-the-switchover)
- [Step 5: Confirm the Cluster Topology](#step-5-confirm-the-cluster-topology)
- [Step 6: Switchover Confirmation](#step-6-switchover-confirmation)
- [Step 7: Verify Post-Switchover Status](#step-7-verify-post-switchover-status)

---

## Step 1: Initiate the Switchover

Run the `switchover` operation using the `patronictl` command-line utility:

```bash
patronictl -c /etc/patroni/patroni.yml switchover
```

> **Note:** Every `patronictl` command requires a YAML configuration file that contains the cluster information.

---

## Step 2: Confirm the Current Primary

`patronictl` detects the current primary server and prompts for confirmation:

```
Primary [pg0]:
```

Press **Enter** to accept the detected primary, or type the server name explicitly (`pg0` in this example) and press Enter.

---

## Step 3: Choose a Candidate

All possible promotion candidates are listed:

```
Candidate ['pg1', 'pg2'] []:
```

You can explicitly select a server by typing its name:

```
Candidate ['pg1', 'pg2'] []: pg1
```

Or simply press **Enter** to let Patroni choose one automatically.

---

## Step 4: Schedule the Switchover

Since a switchover is a planned activity, `patronictl` lets you schedule when it should occur:

```
When should the switchover take place (e.g. 2021-04-20T05:17 )  [now]:
```

The default is immediate (`now`) — press **Enter** to accept it.

---

## Step 5: Confirm the Cluster Topology

Patroni displays the current cluster topology and asks for final confirmation:

```
Current cluster topology
+ Cluster: stampede (6953087616929054699) -----------+
| Member | Host | Role    | State   | TL | Lag in MB |
+--------+------+---------+---------+----+-----------+
| pg0    | pg0  | Leader  | running |  1 |           |
| pg1    | pg1  | Replica | running |  1 |         0 |
| pg2    | pg2  | Replica | running |  1 |         0 |
+--------+------+---------+---------+----+-----------+
Are you sure you want to switchover cluster stampede, demoting current leader pg0? [y/N]: y
```

---

## Step 6: Switchover Confirmation

Patroni performs the switchover and prints a confirmation message along with the new topology:

```
2021-04-20 04:17:51.07301 Successfully switched over to "pg2"
+ Cluster: stampede (6953087616929054699) -----------+
| Member | Host | Role    | State   | TL | Lag in MB |
+--------+------+---------+---------+----+-----------+
| pg0    | pg0  | Replica | stopped |    |   unknown |
| pg1    | pg1  | Replica | running |    |   unknown |
| pg2    | pg2  | Leader  | running |  1 |           |
+--------+------+---------+---------+----+-----------+
```

> **Note:** It may take a few moments for cluster-wide information to settle — wait a few seconds before proceeding.

---

## Step 7: Verify Post-Switchover Status

Confirm the cluster status using the `list` option:

```bash
patronictl -c /etc/patroni/patroni.yml list
```

```
+ Cluster: stampede (6953087616929054699) -----------+
| Member | Host | Role    | State   | TL | Lag in MB |
+--------+------+---------+---------+----+-----------+
| pg0    | pg0  | Replica | running |  2 |         0 |
| pg1    | pg1  | Replica | running |  2 |         0 |
| pg2    | pg2  | Leader  | running |  2 |           |
+--------+------+---------+---------+----+-----------+
```

On a successful switchover, the timeline (`TL`) is incremented, and there is no lag between the leader and its replicas.
