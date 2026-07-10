# Patroni: Connection Routing Using HAProxy

*Author: Rijavan Inamdar*

---

## Introduction

### Traditional vs. Patroni-Assisted Configuration

The traditional, generic configuration of HAProxy with PostgreSQL requires `xinetd` to run a daemon on each cluster node, along with a custom shell script to determine the health of the PostgreSQL instance on each node. This generic configuration is complex to set up, and the shell script repeatedly connects to PostgreSQL to run status-check queries — adding load to the server.

On top of that, this approach can:

- Fill up PostgreSQL logs with connection/disconnection records if `log_connections` and `log_disconnections` are enabled
- Skew monitoring metrics due to the repeated logins and queries

Fortunately, Patroni offers a **REST API on port 8008**, allowing HAProxy to read each node's status directly from the Patroni service. This is both easier to set up and far less resource-intensive, since there's no need to connect to and query the PostgreSQL instances directly — Patroni simply responds with a JSON message to HAProxy's HTTP request, which HAProxy then uses to maintain its routing table.

> ⚠️ **Important:** Older versions of Patroni have been found to respond with a broken HTTP header. Patroni versions **above 1.5** are recommended.

## Table of Contents

- [Configuration Steps](#configuration-steps)
- [Verify Connectivity](#verify-connectivity)
- [Conclusion](#conclusion)

---

## Configuration Steps

### 1. Install HAProxy

On RHEL/CentOS:

```bash
sudo yum install -y haproxy
```

### 2. Configure HAProxy

Edit the configuration file:

```bash
sudo vi /etc/haproxy/haproxy.cfg
```

**Sample configuration:**

```ini
global
    maxconn 100

defaults
    log     global
    mode    tcp
    retries 2
    timeout client  30m
    timeout connect 4s
    timeout server  30m
    timeout check   5s

listen stats
    mode http
    bind *:7000
    stats enable
    stats uri /

listen primary
    bind *:5000
    option httpchk OPTIONS /primary
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server pg0 pg0:5432 maxconn 100 check port 8008
    server pg1 pg1:5432 maxconn 100 check port 8008
    server pg2 pg2:5432 maxconn 100 check port 8008

listen standbys
    bind *:5001
    option httpchk OPTIONS /replica
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server pg0 pg0:5432 maxconn 100 check port 8008
    server pg1 pg1:5432 maxconn 100 check port 8008
    server pg2 pg2:5432 maxconn 100 check port 8008
```

> **Note:** The Patroni role returned by the REST API changed from `master` to `primary` starting with Patroni version 4.0. Refer to the [Patroni health check endpoints documentation](https://patroni.readthedocs.io/en/latest/rest_api.html) for more details.

### How It Works

The sample configuration above defines **two listeners**:

| Listener | Port | Purpose |
|---|---|---|
| `primary` | `5000` | For connections needing read/write (DML) access |
| `standbys` | `5001` | For read-only / reporting connections |

Under each listener, all candidate servers are listed (`pg0`, `pg1`, `pg2` in the sample above). HAProxy checks the status of each server on **port 8008**, where Patroni is listening. Based on the HTTP response header returned, HAProxy updates its internal routing table and redirects connections to the appropriate host.

### 3. Start HAProxy

```bash
sudo systemctl start haproxy
```

---

## Verify Connectivity

### Primary (Read & Write) — Port 5000

Connect and confirm the server is **not** in recovery mode (i.e., it's the primary):

```bash
psql -h localhost -p 5000 -U postgres
```

```sql
postgres=# select pg_is_in_recovery();
 pg_is_in_recovery
--------------------
 f
(1 row)
```

### Standby (Read-Only) — Port 5001

Connect and confirm the server **is** in recovery mode (i.e., it's a standby):

```bash
psql -h localhost -p 5001 -U postgres
```

```sql
postgres=# select pg_is_in_recovery();
 pg_is_in_recovery
--------------------
 t
(1 row)
```

---

## Conclusion

If your cloud provider doesn't offer endpoints with built-in automatic failover, employing software like HAProxy becomes essential. With this setup, an application can simply retry a failed connection, and it will be automatically redirected to a functional database server.

This article demonstrates how straightforward it is to set up HAProxy with a Patroni cluster — notably, **without** relying on any custom scripts or the `xinetd` service.
