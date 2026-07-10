# Patroni: Installation of Patroni Software

*Author: Rijavan Inamdar*

---

## Introduction

Patroni is a high-availability framework/template written in Python. At its core, it's a set of Python scripts (modules) with a handful of dependencies on other general-purpose Python packages. All of these external dependencies are resolved as part of the installation procedure — so installing Patroni really just comes down to installing its dependencies and placing the Patroni-related files in the right location.

Patroni uses an external DCS (Distributed Consensus Store), such as etcd or Consul, for leader election and as a metadata repository. It serves as a unified location for configuration storage, including PostgreSQL-related settings.

This article walks through how to install and set up Patroni, step by step.

> **Note:** Although this article uses Patroni 1.6 and PostgreSQL 11 for illustration, the same procedure applies to more recent versions, such as Patroni 2 with PostgreSQL 12.
>
> Patroni 2 introduces support for PostgreSQL 13 and is the version to focus on for future upgrades — we recommend considering it. It also includes valuable improvements around using `pg_rewind` to sync standby servers, along with new features like support for etcd API v3.
>
> The built-in Raft implementation included in many Patroni versions is deprecated, and we won't be recommending or using it in this article.

## Table of Contents

- [Installing Patroni from the Percona Repository](#installing-patroni-from-the-percona-repository)
- [Using pip to Install Patroni](#using-pip-to-install-patroni)
- [Installing psycopg2](#installing-psycopg2)
- [Installing Patroni Directly from the Source Tarball](#installing-patroni-directly-from-the-source-tarball)
- [Verification of Installation](#verification-of-installation)
- [Tested Versions](#tested-versions)

Patroni can be installed in one of three ways, covered below — starting with the most straightforward.

---

## Installing Patroni from the Percona Repository

Percona Distribution provides ready-made packages for Patroni, which greatly simplifies installation. Unless you have a specific reason to avoid Percona packages, we strongly recommend this approach.

**Step 1 — Add the Percona repository (if not already present):**

```bash
sudo dnf install -y https://repo.percona.com/yum/percona-release-latest.noarch.rpm
```

**Step 2 — Enable the specific PostgreSQL release:**

```bash
sudo percona-release setup ppg-15
```

**Step 3 — Install the Patroni package:**

```bash
sudo dnf install percona-patroni
```

> **Note:** On some systems, you may need to enable the EPEL repository to satisfy dependencies:
> ```bash
> sudo dnf -y install epel-release
> ```

**Step 4 — Install the etcd package:**

```bash
sudo dnf install -y etcd
```

**Step 5 — Install the Python etcd bindings:**

```bash
sudo dnf install -y python3-python-etcd.noarch
```

> On older RHEL-derivative versions, use `yum` instead of `dnf`.

For more details, refer to the [Percona Distribution for PostgreSQL documentation](https://docs.percona.com/postgresql/17/index.html).

---

## Using pip to Install Patroni

The following steps apply to CentOS/RHEL distributions, and can be easily adapted for others.

```bash
sudo dnf install -y python3-pip python3-devel gcc
sudo dnf install -y pyOpenSSL
```

### Notes on Dependencies

- Installing `psutil` requires compilation — this is why the Python 3 development package and GCC compiler are needed. The same applies if `psycopg2` is built from source rather than installed as a binary.
- Some distributions use explicit `python2`/`python3` naming. In such cases, you may need `python3-pip` (as above) or `pip3`:
  ```bash
  sudo dnf install -y pip3   # On some systems, the package name is python3-pip-wheel
  ```
- Older CentOS/RHEL versions might require installing the EPEL repository before `python-pip`:
  ```bash
  sudo yum install -y epel-release
  sudo yum install -y python-pip
  ```
- Alternatively, `pip` can be installed via Python's `setuptools`:
  ```bash
  sudo yum -y install pyOpenSSL python-setuptools.noarch
  sudo easy_install pip
  ```
  > `easy_install` is deprecated, but may still be useful on older systems.
- The GCC compiler and Python development libraries are still required in this case:
  ```bash
  sudo yum install -y python3-devel gcc
  ```

### Upgrading pip

If `pip` was already installed, your existing version (or your distribution's bundled version) may be outdated. We recommend upgrading before installing further packages:

```bash
sudo pip3 install --upgrade pip
sudo pip3 install --upgrade setuptools
sudo pip3 install psycopg2-binary
```

---

## Installing psycopg2

Starting from release 2.8, the binary version of `psycopg2` is no longer installed by default. Installing from source requires a C compiler, plus the PostgreSQL and Python development packages. There are two main installation options, covered below.

### Option A: Install the psycopg2 Binary

This is the easiest way to install `psycopg2`:

```bash
sudo pip3 install psycopg2-binary
```

> ⚠️ The binary installation of `psycopg2` can lead to difficult-to-trace issues, including server crashes (details are documented in the [psycopg2 wiki](https://github.com/psycopg/psycopg2/issues/543)). For this reason, it's discouraged for production servers, though it's fine to use in non-critical environments.

### Option B: Install psycopg2 from Source (Recommended for Production)

The example below uses PostgreSQL 11 — adjust the version number to match your system.

```bash
sudo yum install postgresql11-devel
```

Ensure the full path to `pg_config` is included in your `PATH`:

```bash
PATH=$PATH:/usr/pgsql-11/bin
export PATH
```

Then install `psycopg2` from source:

```bash
sudo PATH=$PATH pip3 install psycopg2
```

This downloads the source tarball, compiles it, and installs it:

```
Collecting psycopg2
  Downloading https://files.pythonhosted.org/packages/84/d7/6a93c99b5ba4d4d22daa3928b983cec66df4536ca50b22ce5dcac65e4e71/psycopg2-2.8.4.tar.gz (377kB)
    100% |████████████████████████████████| 378kB 2.3MB/s
Installing collected packages: psycopg2
  Running setup.py install for psycopg2 ... done
Successfully installed psycopg2-2.8.4
```

### Install Patroni via pip

With the requirements in place, Patroni's latest package can be installed directly from pip. The general format is `pip3 install patroni[dependencies]`. If etcd will be used as the DCS:

```bash
sudo pip3 install patroni[etcd]
```

---

## Installing Patroni Directly from the Source Tarball

As an alternative to pip, Patroni can be built from source. First, download the latest release from GitHub (the URL below for v1.6.4 is just an example):

```bash
cd $HOME
curl -LO https://github.com/zalando/patroni/archive/v1.6.4.tar.gz
```

Unpack the tarball and rename the resulting folder to `patroni`:

```bash
tar -xvf v1*
mv patroni-*/ patroni
```

Install the dependencies listed in the requirements file:

```bash
cd patroni/
sudo pip3 install -r requirements.txt
```

This prints output detailing every prerequisite package it installs:

```
Collecting urllib3!=1.21,>=1.19.1 (from -r requirements.txt (line 1))
  Downloading https://files.pythonhosted.org/packages/e8/74/6e4f91745020f967d09332bb2b8b9b10090957334692eb88ea4afe91b77f/urllib3-1.25.8-py2.py3-none-any.whl (125kB)
    100% |████████████████████████████████| 133kB 3.1MB/s
Collecting boto (from -r requirements.txt (line 2))
  Downloading https://files.pythonhosted.org/packages/23/10/c0b78c27298029e4454a472a1919bde20cb182dab1662cec7f2ca1dcc523/boto-2.49.0-py2.py3-none-any.whl (1.4MB)
    100% |████████████████████████████████| 1.4MB 774kB/s
Collecting PyYAML (from -r requirements.txt (line 3))
...
Installing collected packages: urllib3, boto, PyYAML, six, kazoo, dnspython, python-etcd, chardet, certifi, idna, requests, python-consul, click, prettytable, pytz, tzlocal, python-dateutil, psutil, cdiff, cachetools, pyasn1, rsa, pyasn1-modules, google-auth, websocket-client, oauthlib, requests-oauthlib, kubernetes
  Running setup.py install for PyYAML ... done
  Running setup.py install for python-etcd ... done
  Running setup.py install for prettytable ... done
  Running setup.py install for psutil ... done
  Running setup.py install for cdiff ... done
Successfully installed PyYAML-5.3.1 boto-2.49.0 cachetools-4.0.0 cdiff-1.0 certifi-2019.11.28 chardet-3.0.4 click-7.1.1 dnspython-1.16.0 google-auth-1.11.3 idna-2.9 kazoo-2.7.0 kubernetes-10.0.1 oauthlib-3.1.0 prettytable-0.7.2 psutil-5.7.0 pyasn1-0.4.8 pyasn1-modules-0.2.8 python-consul-1.1.0 python-dateutil-2.8.1 python-etcd-0.4.5 pytz-2019.3 requests-2.23.0 requests-oauthlib-1.3.0 rsa-4.0 six-1.14.0 tzlocal-2.0.0 urllib3-1.25.8 websocket-client-0.57.0
```

Now install Patroni itself:

```bash
sudo python3 setup.py install
```

Example output:

```
running install
running bdist_egg
running egg_info
creating patroni.egg-info
writing patroni.egg-info/PKG-INFO
writing dependency_links to patroni.egg-info/dependency_links.txt
writing entry points to patroni.egg-info/entry_points.txt
writing requirements to patroni.egg-info/requires.txt
writing top-level names to patroni.egg-info/top_level.txt
writing manifest file 'patroni.egg-info/SOURCES.txt'
reading manifest file 'patroni.egg-info/SOURCES.txt'
reading manifest template 'MANIFEST.in'
writing manifest file 'patroni.egg-info/SOURCES.txt'
installing library code to build/bdist.linux-x86_64/egg
running install_lib
running build_py
creating build
creating build/lib
creating build/lib/patroni
....
Using /usr/local/lib/python3.6/site-packages
Finished processing dependencies for patroni==1.6.4
```

This installs Patroni and verifies all of its dependencies.

---

## Verification of Installation

**Check the installed Patroni version:**

```bash
$ patroni --version
patroni 1.6.4
```

**Check the location of the Patroni executable:**

```bash
$ which patroni
/usr/local/bin/patroni
```

**Confirm it's running under Python 3** by checking the shebang line of the executable:

```bash
$ head /usr/local/bin/patroni
```

```
#!/bin/python3
# EASY-INSTALL-ENTRY-SCRIPT: 'patroni==1.6.4','console_scripts','patroni'
__requires__ = 'patroni==1.6.4'
...
```

---

## Tested Versions

This article has been tested against the following versions:

| Component | Version |
|---|---|
| PostgreSQL | 11 and above |
| Patroni | 1.5 or above |
