# Patroni: Installation on Ubuntu

*Author: Rijavan Inamdar*

---

## Introduction

This article details the Ubuntu-specific instructions for installing Patroni. For general installation guidance, refer to [Patroni: Installation of Patroni Software](#).

This article is a supplement to that main guide, intended to walk through the environment-specific steps needed on Ubuntu.

## Table of Contents

- [Preparing for Installation](#preparing-for-installation)
- [Installing Patroni from the Percona Repository](#installing-patroni-from-the-percona-repository)
- [Installing Patroni from pip](#installing-patroni-from-pip)
- [Post-Installation Configuration and Bootstrapping](#post-installation-configuration-and-bootstrapping)

---

## Preparing for Installation

Patroni uses the PostgreSQL binaries already present on the system. This article assumes PostgreSQL binaries are already installed.

If they aren't, install your desired PostgreSQL version from either the [Percona repository](https://docs.percona.com/percona-software-repositories/installing.html) or the [PostgreSQL Global Development Group's (PGDG) repository](https://www.postgresql.org/download/linux/ubuntu/).

### Disabling the systemd Service for PostgreSQL

After installing PostgreSQL, stop and disable its systemd service so that it isn't brought up independently by systemd — in a Patroni cluster, PostgreSQL instances must always be started **by Patroni**.

> If a usable database already exists, you don't need to stop the PostgreSQL service — Patroni can take over a running instance. However, disabling the systemd service is still **mandatory**, to guarantee PostgreSQL is always started by Patroni rather than by systemd.

```bash
sudo systemctl stop postgresql@12-main
sudo systemctl disable postgresql@12-main
sudo systemctl disable postgresql
sudo systemctl status postgresql@12-main
```

On Ubuntu, PostgreSQL's systemd service is templatized. Disabling the top-level `postgresql` service — on which the other templatized services depend — is sufficient to ensure no PostgreSQL instance starts without Patroni:

```bash
sudo systemctl disable postgresql
```

### Installing etcd

Install etcd on every node of the cluster, but keep the service stopped until you're ready for post-installation configuration:

```bash
sudo apt install etcd
sudo systemctl stop etcd
```

### Choosing an Installation Method

Beyond installing from source (covered in [Patroni: Installation of Patroni Software](#)), there are two other ways to install Patroni on Ubuntu:

1. **From the Percona repository** — the easiest method.
2. **Using Python pip** — the most widely used method, and one that ensures the most recent version is installed.

---

## Installing Patroni from the Percona Repository

**Step 1 — Download the Percona repository package:**

```bash
wget https://repo.percona.com/apt/percona-release_latest.$(lsb_release -sc)_all.deb
```

**Step 2 — Install the package:**

```bash
sudo dpkg -i percona-release*.deb
```

**Step 3 — Enable the repository for a specific PostgreSQL version:**

```bash
sudo percona-release setup ppg12
```

> **Note:** Patroni is an independent package — the PostgreSQL version specified here is not significant to which Patroni version gets installed.

**Step 4 — Install Patroni:**

```bash
sudo apt install percona-patroni
```

**Step 5 — Verify the installed version:**

```bash
$ patroni --version
patroni 2.0.1
```

---

## Installing Patroni from pip

**Step 1 — Update the package repository:**

```bash
sudo apt update
```

**Step 2 — Install Python pip:**

```bash
sudo apt install -y python3-pip
```

**Step 3 — Install the `python3-psycopg2` package:**

```bash
sudo apt-get install python3-psycopg2
```

**Step 4 — Install Patroni with etcd support:**

```bash
sudo pip3 install patroni[etcd]
```

**Step 5 — Verify the installation:**

```bash
$ patroni --version
patroni 2.0.2
```

---

## Post-Installation Configuration and Bootstrapping

Once Patroni is installed, proceed to [Patroni: Configuration and Bootstrapping](#) for the remaining post-installation configuration and cluster bootstrapping steps.
