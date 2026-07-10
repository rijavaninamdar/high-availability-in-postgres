# Building an EDB Postgres Advanced Server HA Stack: EFM, PEM, and Barman on RHEL 8.2

*Author: Rijavan Inamdar*

---

## Introduction

This article documents an end-to-end lab build of a highly available EDB Postgres Advanced Server (EPAS) 15 cluster on RHEL 8.2, combining:

- **EFM (Failover Manager)** for automated failover
- **PEM (Postgres Enterprise Manager)** for monitoring
- **Barman** for backup and disaster recovery

It also captures real errors encountered along the way and how each was resolved — useful as a troubleshooting reference for similar deployments.

## Table of Contents

- [Environment and Architecture](#environment-and-architecture)
- [1. RHEL Subscription Registration](#1-rhel-subscription-registration)
- [2. Installing EDB Advanced Server 15](#2-installing-edb-advanced-server-15)
- [3. Setting Up Streaming Replication](#3-setting-up-streaming-replication)
- [4. Configuring EFM (Failover Manager)](#4-configuring-efm-failover-manager)
- [5. Installing and Configuring PEM](#5-installing-and-configuring-pem)
- [6. Installing and Configuring Barman](#6-installing-and-configuring-barman)
- [7. Taking Incremental Backups](#7-taking-incremental-backups)
- [8. Incident Walkthrough: Recovering a Standby from Barman](#8-incident-walkthrough-recovering-a-standby-from-barman)
- [9. Configuring Barman for Multiple Clusters](#9-configuring-barman-for-multiple-clusters)
- [10. Troubleshooting Reference](#10-troubleshooting-reference)

---

## Environment and Architecture

| OS | RHEL 8.2 |
|---|---|

**Nodes:**

| Node | Role |
|---|---|
| `rijavan-node-01` | Primary + EFM + PEM host manager server + Barman packages (to add Barman server into PEM monitoring) |
| `rijavan-node-02` | Standby + EFM + PEM agent |
| `rijavan-node-03` | Witness + EFM + Barman |

---

## 1. RHEL Subscription Registration

After creating a fresh RHEL VM, attempting to install packages before registering the subscription fails:

```
[root@rijavan-node-01 ~]# sudo dnf -y install edb-as15-server
Updating Subscription Management repositories.
Unable to read consumer identity
This system is not registered to Red Hat Subscription Management. You can use subscription-manager to register.
Last metadata expiration check: 0:00:55 ago on Sat 19 Aug 2023 06:47:07 AM EDT.
Error:
 Problem: cannot install the best candidate for the job
  - nothing provides lz4 needed by edb-as15-server-15.3.0-1.el8.x86_64
(try to add '--skip-broken' to skip uninstallable packages or '--nobest' to use not only best candidate packages)
```

**Resolution** — register the system:

```bash
subscription-manager register
```

```
Registering to: subscription.rhsm.redhat.com:443/subscription
Username: Rijavan.inamdar
Password:
The system has been registered with ID: 41cc293c-71e7-49ac-bbe6-6994c52847ca
The registered system name is: rijavan-node-01.novalocal
```

> Perform this registration on **all three nodes** before continuing.

---

## 2. Installing EDB Advanced Server 15

Install `edb-as15-server` on all three machines:

```bash
sudo dnf -y install edb-as15-server
```

Set the password for the `enterprisedb` OS user on all three machines:

```bash
passwd enterprisedb
```

### Initialize the Database (Primary and Standby)

```bash
/usr/edb/as15/bin/edb-as-15-setup initdb
```

On the primary node, set the archive location and configure `archive_mode` / `archive_command`:

```
Archive location: /var/lib/edb/as15/archive
```

### Set the `enterprisedb` Database User Password

```bash
psql -d edb
```

```sql
edb=# \password
Enter new password for user "enterprisedb":
Enter it again:
```

### Configure `pg_hba.conf`

```
# TYPE  DATABASE        USER            ADDRESS                 METHOD

# "local" is for Unix domain socket connections only
local   all             all                                     trust
# IPv4 local connections:
host    all             all             127.0.0.1/32            trust
host    all     all     0.0.0.0/0       md5
# IPv6 local connections:
host    all             all             ::1/128                 md5
# Allow replication connections from localhost, by a user with the
# replication privilege.
local   replication     all                                     trust
host    replication     all             127.0.0.1/32            trust
host    replication all     0.0.0.0/0       md5
host    replication     all             ::1/128                 md5
```

### Set Up Passwordless SSH for the `enterprisedb` User

```bash
ssh-keygen -t rsa
```

```
Generating public/private rsa key pair.
Enter file in which to save the key (/var/lib/edb/.ssh/id_rsa):
Your identification has been saved in /var/lib/edb/.ssh/id_rsa.
Your public key has been saved in /var/lib/edb/.ssh/id_rsa.pub.
```

**Error while copying the key to the standby:**

```
[enterprisedb@rijavan-node-01 .ssh]$ ssh-copy-id enterprisedb@192.168.2.8
...
enterprisedb@192.168.2.8: Permission denied (publickey,gssapi-keyex,gssapi-with-mic).
```

**Resolution** — enable password authentication temporarily on both primary and standby:

```bash
vi /etc/ssh/sshd_config
# Find PasswordAuthentication and set it to:
PasswordAuthentication yes
```

```bash
sudo systemctl restart sshd
```

Retry the key copy — this time it succeeds:

```bash
ssh-copy-id enterprisedb@192.168.2.8
```

```
Number of key(s) added: 1
```

Verify the connection:

```bash
ssh 'enterprisedb@192.168.2.8'
```

---

## 3. Setting Up Streaming Replication

This section sets up asynchronous streaming replication.

### Clear the Data Directory on the Standby

```bash
cd /var/lib/edb/as15/data
rm -rf *
```

### Create the Archive Directory on the Standby

```bash
mkdir /var/lib/edb/as15/archive
chown enterprisedb:enterprisedb /var/lib/edb/as15/archive
chmod 700 /var/lib/edb/as15/archive
```

### Run `pg_basebackup` from the Standby

```bash
/usr/edb/as15/bin/pg_basebackup -h 192.168.2.79 -U enterprisedb -p 5444 -D /var/lib/edb/as15/data/ -Xs -R -P
```

```
71409/71409 kB (100%), 1/1 tablespace
```

### Start EPAS on the Standby

```bash
systemctl start edb-as-15
```

### Verify Replication Status

```sql
db01=# select pg_is_in_recovery();
 pg_is_in_recovery
--------------------
 t
(1 row)

db01=# \x
db01=# select * from pg_stat_wal_receiver;
```

```
-[ RECORD 1 ]---------+---------------------------------------------
pid                   | 16757
status                | streaming
receive_start_lsn     | 0/7000000
written_lsn           | 0/7000060
flushed_lsn           | 0/7000060
sender_host           | 192.168.2.79
sender_port           | 5444
```

Confirm the standby rejects writes:

```sql
db01=# create table t1 (id int);
ERROR:  cannot execute CREATE TABLE in a read-only transaction
```

---

## 4. Configuring EFM (Failover Manager)

### Install EFM on All Three Nodes

```bash
sudo dnf -y install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
sudo dnf -y install edb-efm47
```

### Prepare the Configuration Files

```bash
cd /etc/edb/efm-4.7/
cp efm.properties.in efm.properties
chown efm:efm efm.properties
cp efm.nodes.in efm.nodes
chown efm:efm efm.nodes
chmod 0666 efm.properties efm.nodes
```

### Configure `efm.properties` (Primary Node)

```ini
db.user=enterprisedb
db.password.encrypted=
db.port=5444
db.database=edb

db.service.owner=enterprisedb
db.service.name=edb-as-15
db.bin=/usr/edb/as15/bin
db.data.dir=/var/lib/edb/as15/data
db.config.dir=/var/lib/edb/as15/data
user.email=rijavan.inamdar@enterprisedb.com
from.email=efm@Rijavan-node-01@234
notification.level=INFO
bind.address=192.168.2.79:7800
admin.port=7809
is.witness=false
local.period=10
local.timeout=60
local.timeout.final=10
remote.timeout=10
node.timeout=50
primary.shutdown.as.failure=true
ping.server.ip=8.8.8.8
ping.server.command=/bin/ping -q -c3 -w5
auto.allow.hosts=true
auto.failover=true
auto.reconfigure=true
promotable=true
```

### Encrypt the Database Password

```bash
cd /usr/edb/efm-4.7/bin
./efm encrypt efm
```

**Error:**

```
which: no java in (/usr/local/bin:/usr/bin:/usr/local/sbin:/usr/sbin)
Unable to find JRE in path.
```

**Resolution** — EFM is Java-based, so Java must be installed on every node:

```bash
dnf install java
```

Retry:

```bash
sudo su - efm
cd /usr/edb/efm-4.7/bin/
./efm encrypt efm
```

```
Please enter the password and hit enter:
Please enter the password again to confirm:

The encrypted password is: d6cad98d315dfbda601139d2c1b49068

Please paste this into your efm.properties file
    db.password.encrypted=d6cad98d315dfbda601139d2c1b49068
```

> `efm encrypt` reads the `efm.properties` file and encrypts against the user specified in `db.user`.

Add the encrypted password to `efm.properties`:

```ini
db.password.encrypted=d6cad98d315dfbda601139d2c1b49068
```

### Start EFM on the Primary

```bash
systemctl start edb-efm-4.7
```

### Check Cluster Status

```bash
/usr/edb/efm-4.7/bin/efm cluster-status efm
```

```
Cluster Status: efm

    Agent Type  Address              DB       VIP
    ----------------------------------------------------------------
    Primary     192.168.2.79         UP

Allowed node host list:
    192.168.2.79

Membership coordinator: 192.168.2.79
```

### Add the Standby and Witness to the EFM Cluster

Add all node addresses to `efm.nodes`:

```
# List of node address:port combinations separated by whitespace.
# The list should include at least the membership coordinator's address.
192.168.2.79:7800
192.168.2.8:7800
192.168.2.83:7800
```

Transfer the configuration files to the standby and witness:

```bash
scp efm.properties efm.nodes root@192.168.2.8:/etc/edb/efm-4.7/
scp efm.properties efm.nodes root@192.168.2.83:/etc/edb/efm-4.7/
```

**On the standby**, update `efm.properties`:

```ini
bind.address=192.168.2.8:7800
from.email=efm@Rijavan-node-02@235
```

**On the witness**, update `efm.properties`:

```ini
bind.address=192.168.2.83:7800
from.email=efm@Rijavan-node-03@233
is.witness=true
```

### Allow the New Nodes (from the Primary)

```bash
/usr/edb/efm-4.7/bin/efm allow-node efm 192.168.2.8
/usr/edb/efm-4.7/bin/efm allow-node efm 192.168.2.83
```

### Start EFM on the Standby and Witness

**Error on start:**

```
2023-08-19 09:41:18 File is not writable: /etc/edb/efm-4.7/efm.nodes
```

**Resolution** — fix file ownership and permissions (perform on both standby and witness):

```bash
chown efm:efm efm.nodes efm.properties
chmod 0666 efm.nodes efm.properties
```

Start EFM:

```bash
systemctl start edb-efm-4.7
```

### Verify the Full Cluster

```bash
/usr/edb/efm-4.7/bin/efm cluster-status efm
```

```
Cluster Status: efm

    Agent Type  Address              DB       VIP
    ----------------------------------------------------------------
    Primary     192.168.2.79         UP
    Standby     192.168.2.8          UP
    Witness     192.168.2.83         N/A

Allowed node host list:
    192.168.2.79 192.168.2.8 192.168.2.83

Membership coordinator: 192.168.2.79

Standby priority host list:
    192.168.2.8

Promote Status:

    DB Type     Address              WAL Received LSN   WAL Replayed LSN   Info
    ---------------------------------------------------------------------------
    Primary     192.168.2.79                            0/7027798
    Standby     192.168.2.8          0/7027798           0/7027798

    Standby database(s) in sync with primary. It is safe to promote.
```

The standby and witness are now part of the EFM cluster family.

---

## 5. Installing and Configuring PEM

### Install PEM Server (Primary)

```bash
sudo dnf -y install edb-pem
```

### Run the Configuration Script

```bash
/usr/edb/pem/bin/configure-pem-server.sh
```

```
Install type: 1:Web Services and Database, 2:Web Services 3: Database [ ] :1
Enter local database server installation path [ ] :/usr/edb/as15
Enter database super user name [ ] :enterprisedb
Enter database server port number [ ] :5444
Please enter CIDR formatted network address range [ 0.0.0.0/0 ] :0.0.0.0/0
Enter database systemd unit file or init script name [ ] :edb-as-15
Please specify agent certificate path [ /root/.pem/ ] :
```

**Error:**

```
[Error] The SSLUtils extension was not found in the selected database server.
The latest version of this extension must be installed before the server can be
used by the EDB Postgres Enterprise Manager Server.
```

**Resolution** — install the SSLUtils package for the corresponding server version:

```bash
dnf install edb-as15-server-sslutils
```

Re-run the configuration script:

```bash
/usr/edb/pem/bin/configure-pem-server.sh
```

```
Postgres Enterprise Manager Agent registered successfully!
-->  [Info] Configured the webservice for EDB Postgres Enterprise Manager (PEM) Server on port '8443'.
-->  [Info] PEM server can be accessed at https://127.0.0.1:8443/pem at your browser
```

### Access the PEM Web UI

```
https://127.0.0.1:8443/pem
```

> If installed on a cloud VM (e.g., OpenStack) with a floating IP, use the floating IP instead of `127.0.0.1`:
> ```
> https://<floating-ip>:8443/pem
> ```

### Add the Standby to PEM

Install the PEM agent on the standby:

```bash
sudo dnf -y install edb-pem-agent
```

Register the agent with the PEM server:

```bash
/usr/edb/pem/agent/bin/pemworker --register-agent \
  --pem-server 192.168.2.79 \
  --pem-user enterprisedb \
  --pem-port 5444
```

**Error:**

```
ERROR: PEM_SERVER_PASSWORD environment variable is not set
```

**Resolution** — set the environment variable before registering:

```bash
export PEM_SERVER_PASSWORD=edb
```

Re-run the registration command:

```
Postgres Enterprise Manager Agent registered successfully!
```

**Agent showed "unknown" status in the dashboard** — resolved by restarting the agent after registration:

```bash
sudo systemctl daemon-reload
sudo systemctl restart pemagent
```

### Add Replication Details to the PEM Dashboard

In the PEM dashboard, navigate to the added server's **Properties → Advanced** tab and add the EFM cluster details. After reloading, replication details become visible under **Streaming Replication**.

---

## 6. Installing and Configuring Barman

### Install Barman (Witness Node)

```bash
sudo dnf -y install barman
```

### Install the Barman CLI (Primary and Standby)

```bash
sudo dnf -y install barman-cli
```

### Set Up Passwordless SSH: Barman → PostgreSQL Hosts

Set the `barman` OS user's password and generate an SSH key:

```bash
passwd barman
su - barman
ssh-keygen -t rsa
```

Copy the key to the primary:

```bash
ssh-copy-id -i enterprisedb@192.168.2.79
```

Copy the primary's key to the witness (Barman host), and repeat from the standby:

```bash
ssh-copy-id -i barman@192.168.2.83
```

Verify each connection with a test `ssh` login/logout.

### Create the Barman Database User

On the primary:

```sql
edb=# create user barman with superuser password 'barman';
CREATE ROLE
```

### Update `pg_hba.conf`

Rather than restricting to the Barman host's IP specifically, `trust` was configured broadly for `0.0.0.0/0` on both the general and replication sections (adjust to your security requirements):

```
# TYPE  DATABASE        USER            ADDRESS                 METHOD
local   all             all                                     trust
host    all             all             127.0.0.1/32            trust
host    all     all     0.0.0.0/0       trust
host    all             all             ::1/128                 md5
local   replication     all                                     trust
host    replication     all             127.0.0.1/32            trust
host    replication all     0.0.0.0/0       trust
host    replication     all             ::1/128                 md5
host    all             enterprisedb             192.168.2.79/32            md5
```

> Apply the same `pg_hba.conf` changes on the standby.

### Configure `archive_command` (Primary and Standby)

```ini
archive_command = 'cp %p /var/lib/edb/as15/archive/%f && barman-wal-archive 192.168.2.83 EPAS15 %p'
```

Reload PostgreSQL to apply:

```bash
systemctl reload edb-as-15
```

### Verify Connectivity from Barman to the Database Hosts

```bash
/usr/edb/as15/bin/psql -c 'SELECT version();' -U barman -h 192.168.2.79 -d edb
/usr/edb/as15/bin/psql -c 'SELECT version();' -U barman -h 192.168.2.8 -d edb
```

Check replication access:

```bash
/usr/edb/as15/bin/psql -c 'IDENTIFY_SYSTEM;' -U barman -h 192.168.2.79 replication=1;
```

```
      systemid       | timeline |  xlogpos   | dbname
---------------------+----------+------------+--------
 7268988418399183072 |        1 | 0/40006ED8 |
```

### Configure `.pgpass`

**On the Barman host:**

```bash
vi /var/lib/barman/.pgpass
```

```
*:5444:*:barman:barman
```

```bash
chmod 0600 /var/lib/barman/.pgpass
```

**On both the primary and standby:**

```
*:5444:*:barman:barman
*:5444:*:enterprisedb:edb
```

```bash
chmod 0600 .pgpass
```

### Configure `barman.conf`

```ini
barman_user = barman
barman_home = /var/lib/barman
log_file = /var/log/barman/barman.log
parallel_jobs = 10
immediate_checkpoint = true
```

### Configure the Server File (`EPAS15.conf`)

```bash
cd /etc/barman.d/
cp streaming-server.conf-template EPAS15.conf
vi EPAS15.conf
```

```ini
[EPAS15]
description = "Example of PostgreSQL Database (Streaming-Only)"

conninfo = host=192.168.2.79 user=barman dbname=edb

streaming_conninfo = host=192.168.2.79 user=barman

backup_method = postgres

streaming_archiver = on
slot_name = barman15
create_slot = auto
streaming_archiver_name = barman_receive_wal
archiver = on

path_prefix = "/usr/edb/as15/bin"
```

### Run `barman check`

```bash
sudo su - barman
barman check EPAS15
```

```
Server EPAS15:
    PostgreSQL: OK
    superuser or standard user with backup privileges: OK
    PostgreSQL streaming: OK
    wal_level: OK
    replication slot: OK
    directories: OK
    ...
    archiver errors: OK
```

Confirm the `pg_receivewal` process is running:

```bash
ps -ef | grep rece
```

> **Note:** If the receiver process isn't started on the Barman host, run:
> ```bash
> barman cron
> ```

### Take a Base Backup

```bash
barman backup EPAS15
```

```
Starting backup using postgres method for server EPAS15 in /var/lib/barman/EPAS15/base/20230820T071552
Backup start at LSN: 0/42676958 (000000010000000000000042, 00676958)
Starting backup copy via pg_basebackup for 20230820T071552
Copy done (time: 12 seconds)
Backup size: 186.5 MiB
Backup end at LSN: 0/44004188 (000000010000000000000044, 00004188)
Backup completed (start time: 2023-08-20 07:15:52.456061, elapsed time: 13 seconds)
```

### List and Inspect Backups

```bash
barman list-backup EPAS15
```

```
EPAS15 20230820T071552 - Sun Aug 20 07:16:04 2023 - Size: 218.5 MiB - WAL Size: 0 B
```

```bash
barman show-backup EPAS15 20230820T071552
```

```
Backup 20230820T071552:
  Server Name            : EPAS15
  System Id              : 7268988418399183072
  Status                 : DONE
  PostgreSQL Version     : 150003
  PGDATA directory       : /var/lib/edb/as15/data
  ...
  Catalog information:
    Retention Policy     : VALID
    Previous Backup      : - (this is the oldest base backup)
    Next Backup          : - (this is the latest base backup)
```

### Integrating the Barman Host into the PEM Dashboard

Reference: [Monitoring Barman — EDB PEM Documentation](https://www.enterprisedb.com/docs/pem/latest/monitoring_barman/)

#### Install the Postgres Backup API

```bash
sudo yum install pg-backup-api
```

```bash
systemctl start pg-backup-api
systemctl enable pg-backup-api
```

Check status:

```bash
systemctl status pg-backup-api
```

Verify the API is serving on port 7480:

```bash
curl http://localhost:7480/status
```

```
"OK"
```

#### Secure the API with Apache (httpd)

Reference: [Securing Postgres Backup API — EDB Documentation](https://www.enterprisedb.com/docs/supported-open-source/barman/pg-backup-api/02-securing-pg-backup-api/)

```bash
dnf install httpd mod_ssl
```

Check the `Listen` directive:

```bash
cat /etc/httpd/conf/httpd.conf | grep Listen
```

Start Apache:

```bash
systemctl start httpd
```

Disable plain port 80 listening, keeping only SSL:

```bash
sudo sed -i 's/^Listen 80$/#Listen 80/' /etc/httpd/conf/httpd.conf
systemctl restart httpd
systemctl enable httpd
```

#### Install the PEM Agent on the Barman Host

```bash
sudo dnf -y install edb-pem-agent
```

```bash
cd /usr/edb/pem/agent/etc/
cp agent.cfg.sample agent.cfg
```

`agent.cfg` template:

```ini
[PEM/agent]
pem_host=<HOST>
pem_port=<PORT>
agent_id=<AGENT ID>
agent_ssl_key=<PATHNAME OF AGENT KEY>
agent_ssl_crt=<PATHNAME OF AGENT CRT>
log_level=warning
log_location=/var/log/worker.log
agent_log_location=/var/log/agent.log
long_wait=30
short_wait=10
alert_threads=0
enable_smtp=false
enable_snmp=false
enable_webhook=false
max_webhook_retries=3
allow_server_restart=true
max_connections=0
connection_lifetime=0
```

#### Register the Barman Agent

**Error:**

```bash
/usr/edb/pem/agent/bin/pemworker --register-barman --api-url http://localhost:7480 --description EPAS15-barman -c /usr/edb/pem/agent/etc/agent.cfg
```

```
ERROR: pem_port must be an integer between 0 and 65535
```

**Resolution — start the PEM agent service**, which hadn't been started yet:

```bash
systemctl start pemagent
```

**Second error**, seen in `/var/log/worker.log`:

```
WARNING: Disabling the execution of Job steps (with shell script) & batch probes as no user specified to run the shell scripts.
Please specify 'batch_script_user' configuration option in the agent configuration file.
WARNING: ConnectToPEM: unable to connect to PEM database: connection to server at "192.168.2.79", port 8443 failed: server closed the connection unexpectedly
```

**Resolution** — correct the `pem_port` and other values in `agent.cfg`:

```ini
[PEM/agent]
pem_host=192.168.2.79
pem_port=5444
agent_id=1
log_level=warning
log_location=/var/log/worker.log
agent_log_location=/var/log/agent.log
long_wait=30
short_wait=10
alert_threads=0
enable_smtp=false
enable_snmp=false
enable_webhook=true
max_webhook_retries=3
allow_server_restart=true
max_connections=0
connection_lifetime=0
```

Re-run the registration:

```bash
/usr/edb/pem/agent/bin/pemworker --register-barman --api-url http://localhost:7480 --description EPAS15-barman -c /usr/edb/pem/agent/etc/agent.cfg
```

```
Barman API successfully registered!
BARMAN ID: 3

** NOTE: Please restart the pemAgent to take these changes in effect.
```

```bash
systemctl restart pemagent
```

The Barman server is now visible in the PEM dashboard.

---

## 7. Taking Incremental Backups

Update the `EPAS15.conf` file to switch to rsync-based incremental backups:

```ini
backup_method = rsync
reuse_backup = link
backup_options = concurrent_backup
ssh_command = ssh enterprisedb@192.168.2.79
```

Re-check and take a backup:

```bash
barman check EPAS15
barman backup EPAS15
```

```
Starting backup using rsync-concurrent method for server EPAS15 in /var/lib/barman/EPAS15/base/20230820T120043
Backup start at LSN: 0/590000D8 (000000010000000000000059, 000000D8)
Starting backup copy via rsync/SSH for 20230820T120043 (10 jobs)
Copy done (time: 4 seconds)
Backup size: 198.7 MiB. Actual size on disk: 198.7 MiB (-0.00% deduplication ratio).
Backup completed (start time: 2023-08-20 12:00:44.174192, elapsed time: 11 seconds)
```

---

## 8. Incident Walkthrough: Recovering a Standby from Barman

### Scenario

All servers experienced a power loss. After restarting, the EPAS service on the standby did not come back up automatically. A few days later, the standby began failing with:

```
2023-08-25 09:51:33 EDT FATAL:  could not receive data from WAL stream: ERROR:  requested WAL segment 000000010000000100000049 has already been removed
```

The requested WAL segment had already been removed from the primary's archive location — but since Barman had been continuously archiving from the primary, that file (and subsequent ones) should still exist there.

### Confirm the WAL File Exists on the Barman Server

```bash
ls -lrth /var/lib/barman/EPAS15/wals/0000000100000001/000000010000000100000049
```

```
-rw-------. 1 barman barman 16M Aug 24 04:01 000000010000000100000049
```

The file was present — so a Barman-based recovery was viable.

### Perform the Recovery

Stop EPAS services on the standby before recovering.

Confirm at least one usable backup exists:

```bash
barman list-backup EPAS15
```

```
EPAS15 20230820T120043 - Sun Aug 20 12:00:51 2023 - Size: 214.7 MiB - WAL Size: 4.9 GiB
EPAS15 20230820T114039 - Sun Aug 20 11:40:51 2023 - Size: 230.5 MiB - WAL Size: 48.0 MiB
```

Run the recovery:

```bash
barman recover --remote-ssh-command 'ssh enterprisedb@192.168.2.8' EPAS15 20230820T120043 /var/lib/edb/as15/data
```

```
Starting remote restore for server EPAS15 using backup 20230820T120043
Copying the base backup.
Copying required WAL segments.
Generating archive status files
Identify dangerous settings in destination directory.

IMPORTANT
These settings have been modified to prevent data losses

postgresql.conf line 264: archive_command = false

WARNING
You are required to review the following options as potentially dangerous

postgresql.conf line 105: ssl_ca_file = 'root.crt'
postgresql.conf line 106: ssl_cert_file = 'server.crt'
postgresql.conf line 107: ssl_crl_file = 'root.crl'
postgresql.conf line 109: ssl_key_file = 'server.key'

Recovery completed (start time: 2023-08-25 10:30:54.417356-04:00, elapsed time: 1 minute, 41 seconds)
Your PostgreSQL server has been successfully prepared for recovery!
```

### Restore the `archive_command`

Barman disables `archive_command` by default for safety during recovery:

```ini
#BARMAN#archive_command = 'cp %p /var/lib/edb/as15/archive/%f && barman-wal-archive 192.168.2.83 EPAS15 %p'
archive_command = false
```

Revert it manually:

```ini
archive_command = 'cp %p /var/lib/edb/as15/archive/%f && barman-wal-archive 192.168.2.83 EPAS15 %p'
#archive_command = false
```

Start the cluster.

> ⚠️ **Important:** Recovering this way brings the node up as a **standalone system**, not automatically as a replica. It's a useful last resort when Barman backups are the only remaining copy of the data.

---

## 9. Configuring Barman for Multiple Clusters

To back up multiple PostgreSQL clusters running on the same server (e.g., a second cluster on a different port), create a second Barman server file following the same pattern — here, `EPAS15_tde.conf`:

```ini
[EPAS15_tde]
description = "Example of PostgreSQL Database (Streaming-Only)"

conninfo = host=192.168.2.79 user=barman dbname=edb port=5448
streaming_conninfo = host=192.168.2.79 user=barman port=5448

backup_method = rsync

streaming_archiver = on
slot_name = barman15_tde
create_slot = auto
streaming_archiver_name = barman_receive_wal
archiver = on
reuse_backup = link
backup_options = concurrent_backup
ssh_command = ssh enterprisedb@192.168.2.79

path_prefix = "/usr/edb/as15/bin"
```

> **Note:** Add a separate port-specific entry in `.pgpass` on both the Barman host and the PostgreSQL host to avoid connection issues:
>
> **Barman host:**
> ```
> *:5444:*:barman:barman
> *:5448:*:barman:barman
> ```
>
> **PostgreSQL host:**
> ```
> *:5444:*:barman:barman
> *:5444:*:enterprisedb:edb
> *:5448:*:barman:barman
> ```

---

## 10. Troubleshooting Reference

The following errors were encountered while running multiple Barman-managed clusters against the same or different PostgreSQL servers.

### Replication Slot Doesn't Exist

```
replication slot: FAILED (replication slot 'barman15_tde' doesn't exist. Please execute 'barman receive-wal --create-slot EPAS15_tde')
```

**Resolution:**

```bash
barman receive-wal --create-slot EPAS15_tde
```

### Slot Not Initialized / `receive-wal` Not Running

```
replication slot: FAILED (slot 'barman15_tde' not initialised: is 'receive-wal' running?)
```

Attempting to start `receive-wal` manually surfaced a starting-point mismatch:

```
EPAS15_tde: pg_receivewal: error: unexpected termination of replication stream: ERROR:  requested starting point 2/0 is ahead of the WAL flush position of this server 0/17019720
```

Further attempts to stop/reset also failed:

```
ERROR: Termination of receive-wal failed: no such process for server EPAS15_tde
ERROR: The receive-wal position is ahead of PostgreSQL current WAL lsn (000000010000000200000000.partial > 000000010000000000000017)
```

**Resolution** — clear the stale streaming directory contents for that server, then let `barman cron` restart the receiver:

```bash
cd /var/lib/barman/EPAS15_tde/streaming/
rm -rf *
barman cron
```

```
Starting WAL archiving for server EPAS15
Starting WAL archiving for server EPAS15_tde
Starting streaming archiver for server EPAS15_tde
```

Confirm both receiver processes are running:

```bash
ps -ef | grep rece
```

### Archiver Errors: Duplicate WAL File

```
archiver errors: FAILED (duplicates: 1)
```

**Resolution** — remove the flagged duplicate file from the server's `errors` directory:

```bash
cd /var/lib/barman/EPAS15/errors/
ls
# 000000010000000100000096.20230825T145202Z.duplicate
rm -rf 000000010000000100000096.20230825T145202Z.duplicate
```

Re-run `barman check` to confirm the error clears.

### Slot Not Initialized, Followed by "WAL Segment Already Removed"

```
replication slot: FAILED (slot 'barman15' not initialised: is 'receive-wal' running?)
```

Attempting a manual `receive-wal` run showed the requested WAL segment was already gone from the primary:

```
EPAS15: could not change directory to "/root": Permission denied
EPAS15: pg_receivewal: error: unexpected termination of replication stream: ERROR:  requested WAL segment 0000000200000002000000C7 has already been removed
```

**Resolution** — reset the `receive-wal` position, which realigns Barman's tracked starting point with the server's current WAL position:

```bash
barman receive-wal --reset EPAS15
```

```
Resetting receive-wal directory status
Removing status file /var/lib/barman/EPAS15/streaming/0000000100000002000000C7.partial
Creating status file /var/lib/barman/EPAS15/streaming/0000000300000002000000CA.partial
```

Confirm with `barman check EPAS15` that `receive-wal running: OK`.
