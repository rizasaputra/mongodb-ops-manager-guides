# Upgrade Ops Manager 8.0 to 9.0

This guide walks you through upgrading the single-host test Ops Manager instance on `opsmgr-1` from the 8.0 series to the 9.0 series, in place, using the RPM package.

> ⚠️ **Testing / evaluation only — not a production configuration.**
> This procedure assumes the single-host lab setup from `00-install-ops-manager.md`:
> one Ops Manager host with no high availability and an Application Database that runs
> as a single `mongod`. On this topology the upgrade is a short outage of the console —
> Ops Manager stops while you swap the package, then starts again. A production,
> highly available Ops Manager (multiple hosts sharing one Application Database) can
> upgrade with no monitoring downtime using **Upgrade Mode**, which does not apply here.
> For production, follow the official
> [Upgrade Ops Manager](https://www.mongodb.com/docs/ops-manager/current/tutorial/upgrade-ops-manager/)
> tutorial.

> **Platform note.** Ops Manager 9.0 adds support for **RHEL 10** (alongside RHEL 9),
> while dropping some older platforms. This project runs on RHEL 9.4, which 9.0 still
> supports, so you can upgrade in place without changing the OS. If you later want to
> move to RHEL 10, treat that as a separate host migration — do not combine an OS upgrade
> with the Ops Manager upgrade in the same window. Always confirm your exact OS against
> the [Ops Manager System Requirements](https://www.mongodb.com/docs/ops-manager/current/core/requirements/)
> before upgrading.

## Overview

You will confirm you are on the latest 8.0 patch, back up the two files Ops Manager
needs to preserve its identity (`conf-mms.properties` and `gen.key`), stop the Ops
Manager service, install the 9.0 RPM over the 8.0 one, start it again, and finally let
Ops Manager push the matching MongoDB Agent update to your managed deployments.

The upgrade replaces only the **Ops Manager Application** on `opsmgr-1`. It does not
change the MongoDB version of any deployment you manage (Project A `new-rs0` or Project B
production). Your managed `mongod` processes keep running throughout — the only thing that
restarts is the Ops Manager Application itself.

Key facts that make this safe on the test host:

- **Managed databases are untouched.** Upgrading the Ops Manager Application does not
  stop, restart, or re-version your managed `mongod`/`mongos`. The console is briefly
  unavailable; the databases are not.
- **No major version may be skipped.** You go `8.0.x` → latest `8.0` patch → latest
  `9.0`. You cannot jump straight from, say, `7.0` to `9.0`; each major release is a hop.
  This guide covers the single hop from 8.0 to 9.0.
- **The backing database must be new enough.** Ops Manager 9.0 requires its Application
  Database to run **MongoDB 6.0 or later**, and the Application Database version must not
  be *later* than the Ops Manager version. This project runs the Application Database on
  MongoDB 8.0 Enterprise, which satisfies both — so upgrading the backing database is
  **optional** (see step 2a if you want it on MongoDB 9.0).

## Prerequisites

- The test Ops Manager instance from `00-install-ops-manager.md`, running an **8.0.x**
  release on `opsmgr-1`, with its Application Database `mongod` healthy on
  `127.0.0.1:27017`. This project runs **RHEL 9.4**, which Ops Manager 9.0 supports;
  9.0 also supports **RHEL 10**, so either is fine.
- `root` (or `sudo`) access on `opsmgr-1`.
- A MongoDB account to download the Ops Manager 9.0 **Enterprise** package.
- HTTPS access from `opsmgr-1` to `downloads.mongodb.com` to pull the RPM (or download
  elsewhere and copy it over, as in the install guide).
- A short maintenance window. The console is down while you swap the package and the
  first 9.0 startup runs its schema migration. Monitoring data gaps during this window
  are expected on a single-host setup.
- Know the current running version. You can read it in the console footer, or on the
  host:

  ```bash
  rpm -q mongodb-mms
  ```

  This prints the installed package version, for example
  `mongodb-mms-8.0.26.500.20260812T1154Z`.

## Steps

Run every step on `opsmgr-1` unless noted. The order matters: back up first, stop the
service, swap the package, start, then update agents.

### 1. Get on the latest 8.0 patch first

Ops Manager upgrades hop one major release at a time, and each hop should start from the
**latest patch** of the current major. Before going to 9.0, make sure you are on the most
recent 8.0.x patch.

First, read the version you are **currently** running. On the host, ask the package
manager:

```bash
rpm -q mongodb-mms
```

This prints something like `mongodb-mms-8.0.26.500.20260812T1154Z`; the `8.0.26` part is
your patch version. You can also read it in the Ops Manager console footer, or at
**Admin → General → Version**.

Now find the **latest** available 8.0 patch to compare against. There is no "check for
updates" button — you compare your version to the published list:

- Open the [Ops Manager 8.0 release notes](https://www.mongodb.com/docs/ops-manager/current/release-notes/application/).
  The newest 8.0.x entry at the top is the latest patch.
- Or open the [Ops Manager release archive](https://www.mongodb.com/try/download/ops-manager/releases/archive)
  and look at the highest 8.0.x build listed for the RHEL 9 rpm.

Compare the two:

- If your `rpm -q` version already matches the latest 8.0.x, continue to step 2.
- If you are on an older 8.0 patch, upgrade to the latest 8.0 patch first using this same
  procedure (steps 2–9), but with the latest **8.0** RPM. Then repeat the procedure for
  9.0. Doing the patch hop first keeps the migration path supported.

> Do not skip majors. If this host were on 7.0, you would upgrade 7.0 → latest 7.0 patch
> → latest 8.0 → latest 9.0, one hop at a time, starting each Ops Manager only after the
> previous hop is up. From 8.0 you only have the single hop to 9.0.

### 2. Confirm the Application Database meets the 9.0 minimum

Ops Manager 9.0 requires the Application Database to run MongoDB **6.0 or later**, and the
Application Database version must not be newer than the Ops Manager version. Confirm the
version of the backing `mongod` on `opsmgr-1`:

```bash
mongosh --quiet --port 27017 --eval 'db.version()'
```

You should see `8.0.x` (this project runs the Application Database on MongoDB 8.0
Enterprise), which is well above the 6.0 minimum. So you are **not required** to touch the
Application Database to upgrade to Ops Manager 9.0 — step 2a below is optional.

### 2a. (Optional) Upgrade the Application Database to MongoDB 9.0 first

> **Optional.** Skip this entire step unless you specifically want the backing database on
> MongoDB 9.0. Ops Manager 9.0 runs fine with the 8.0 Application Database you already
> have. Do this only to keep the backing database current, and do it **before** the Ops
> Manager package upgrade so Ops Manager always starts against a database it supports.

The Application Database on `opsmgr-1` is a single standalone `mongod` you run directly
through systemd (from `00-install-ops-manager.md`) — it is **not** managed by Ops Manager
automation. So you upgrade it by hand, the normal MongoDB way, not through the console.

Two rules shape the order below:

- **One major at a time.** MongoDB upgrades hop one major release per step: 8.0 → 9.0.
  (You cannot skip a major; from 8.0 the only hop is 9.0.)
- **FCV gates the next hop.** Before installing a new major binary, the
  `featureCompatibilityVersion` (FCV) must already be at the current major. Starting a
  9.0 binary against a database whose FCV is still below 8.0 can crash the node.

Because this is the single-host test setup, upgrading the standalone Application Database
means a brief stop of that `mongod` — and Ops Manager cannot run while its backing
database is down. You are already in a maintenance window with Ops Manager about to be
stopped anyway, so sequence it as: stop Ops Manager → upgrade the Application Database →
upgrade Ops Manager.

1. **Stop Ops Manager** so nothing talks to the Application Database while you upgrade it:

   ```bash
   sudo systemctl stop mongodb-mms
   ```

2. **Confirm you are on the latest 8.0 patch and FCV is 8.0.** Check the running version
   and the current FCV:

   ```bash
   mongosh --quiet --port 27017 --eval 'db.version()'
   ```

   ```bash
   mongosh --quiet --port 27017 --eval 'db.adminCommand({getParameter:1, featureCompatibilityVersion:1})'
   ```

   If `featureCompatibilityVersion` is not `8.0`, set it now and confirm it reports back
   as fully upgraded before you change any binary:

   ```bash
   mongosh --quiet --port 27017 --eval 'db.adminCommand({setFeatureCompatibilityVersion:"8.0", confirm:true})'
   ```

3. **Back up the Application Database** before changing the binary. There is no officially
   supported backup method for the Application Database, but a `mongodump` gives you a
   fallback:

   ```bash
   mongodump --port 27017 --out ~/appdb-pre-db-upgrade
   ```

4. **Swap the MongoDB binaries to 9.0.** This project installed the Application Database
   from the MongoDB Enterprise `dnf` repository, so point `dnf` at the 9.0 repo and
   upgrade the package. Create `/etc/yum.repos.d/mongodb-enterprise-9.0.repo`:

   ```ini
   [mongodb-enterprise-9.0]
   name=MongoDB Enterprise Repository
   baseurl=https://repo.mongodb.com/yum/redhat/$releasever/mongodb-enterprise/9.0/$basearch/
   gpgcheck=1
   enabled=1
   gpgkey=https://pgp.mongodb.com/server-9.0.asc
   ```

   Stop the Application Database, upgrade the package, then start it again through the
   same systemd unit so the correct resource limits still apply:

   ```bash
   sudo systemctl stop mongod
   ```

   ```bash
   sudo dnf install -y mongodb-enterprise
   ```

   ```bash
   sudo systemctl start mongod
   ```

5. **Verify the new binary and raise FCV to 9.0.** Confirm `mongod` came back on 9.0:

   ```bash
   mongosh --quiet --port 27017 --eval 'db.version()'
   ```

   Once it reports `9.0.x` and is healthy, enable the 9.0 feature set:

   ```bash
   mongosh --quiet --port 27017 --eval 'db.adminCommand({setFeatureCompatibilityVersion:"9.0", confirm:true})'
   ```

   Confirm the FCV took:

   ```bash
   mongosh --quiet --port 27017 --eval 'db.adminCommand({getParameter:1, featureCompatibilityVersion:1})'
   ```

Leave Ops Manager stopped for now — you will start it on the new 9.0 package in step 8.
The rest of this guide continues the Ops Manager package upgrade; because you already
stopped the service here, you can skip the stop in step 5 when it comes up.

> **Note on managed deployments.** This optional step upgrades only the **Application
> Database** on `opsmgr-1`. It does **not** change the MongoDB version of `new-rs0` or
> production. Those are managed by automation, so you change their version from the
> console (**Deployment → Clusters →** the process's **Version**), following the MongoDB
> release notes — not with `dnf` on the host. That is out of scope for this guide.

### 3. Verify the deployments are healthy before you start

An upgrade is a bad time to discover an unrelated problem. Confirm your managed
deployments are healthy first, so any issue you see afterward is clearly from the upgrade.

- In the console, check that Project A (`new-rs0`) and Project B (production) show green,
  with no pending automation and no stale agents.
- Optionally confirm a managed replica set directly (this does not go through Ops
  Manager):

  ```bash
  mongosh "mongodb://<prod-primary-fqdn>:27017/?replicaSet=<prod-rs-name>" -u <user> -p --quiet --eval 'rs.status().members.map(m => m.stateStr)'
  ```

  - `<prod-primary-fqdn>` — a reachable member of the replica set.
  - `<prod-rs-name>` — the replica set name.
  - `<user>` — a user that can run `rs.status()`.

  Expect one `PRIMARY` and healthy `SECONDARY` members.

### 4. Back up the two files Ops Manager must keep

Ops Manager's identity and its ability to read its own backing data live in two files.
Back them up **before** touching the package, because an upgrade can overwrite or remove
them:

- `conf-mms.properties` — the Ops Manager configuration (central URL, connection string,
  email, and the other settings you set in the install wizard).
- `gen.key` — the key Ops Manager uses to encrypt and decrypt its backing databases and
  stored credentials. Lose this and Ops Manager cannot read its own data.

Copy both to a safe location off the install directory:

```bash
sudo cp -a /opt/mongodb/mms/conf/conf-mms.properties ~/mms-conf-mms.properties.bak
```

```bash
sudo cp -a /opt/mongodb/mms/conf/gen.key ~/mms-gen.key.bak
```

As an extra precaution, you may also take a `mongodump` of the Application Database. There
is no officially supported backup method for the Application Database, but a dump gives
you a fallback if the upgrade fails:

```bash
mongodump --port 27017 --out ~/appdb-predump
```

### 5. Stop the Ops Manager service

Stop the Ops Manager Application so the package can be replaced cleanly. On RHEL 9/10
(systemd):

```bash
sudo systemctl stop mongodb-mms
```

If you did the optional Application Database upgrade in step 2a, Ops Manager is already
stopped — skip this command and continue.

Leave the Application Database `mongod` **running** — you are only stopping the Ops
Manager Application, not its backing database. Confirm the console port stops listening:

```bash
sudo ss -tlnp | grep 8080 || echo "8080 no longer listening"
```

### 6. Download the Ops Manager 9.0 RPM

Get the exact installer URL from the
[Ops Manager release archive](https://www.mongodb.com/try/download/ops-manager/releases/archive).
Find the latest **9.0** release and, under the Red Hat / RHEL (9 or 10) row, copy the
**rpm** link. The file name includes a build stamp you cannot guess, so always copy the
real link rather than typing a version by hand.

Download it directly onto `opsmgr-1`. For example, Ops Manager **9.0.1** is:

```bash
curl -OL "https://downloads.mongodb.com/on-prem-mms/rpm/mongodb-mms-9.0.1.532.20260928T1452Z.x86_64.rpm"
```

- The build stamp (`.532.20260928T1452Z`) is release-specific — use the real link you
  copied from the archive, not a hand-typed version. A made-up version downloads a tiny
  XML error page instead of the RPM, and `rpm -Uvh` then fails with "not an rpm package".
- `-O` keeps the remote file name; `-L` follows redirects.

Verify the download is a real RPM before installing. A failed download is often a small
XML/HTML error page, not the package:

```bash
file mongodb-mms-*.x86_64.rpm
```

It should report something like `RPM v3.0 bin` and be hundreds of MB. If it reports XML or
HTML (a few hundred bytes), the URL was wrong — recopy the link from the archive.

If `opsmgr-1` is air-gapped, download the RPM on another machine and copy it over with
`scp`:

```bash
scp -i <keyfile> mongodb-mms-<9.0-version>.x86_64.rpm <username>@<opsmgr-host>:~
```

### 7. Install the 9.0 package over 8.0

Upgrade the existing package in place with `rpm -U` (upgrade), not `rpm -i` (install).
Use the actual file name you downloaded — type `mongodb-mms-` and press Tab to
autocomplete rather than typing the version by hand:

```bash
sudo rpm -Uvh mongodb-mms-<9.0-version>.x86_64.rpm
```

For example, for Ops Manager 9.0.1:

```bash
sudo rpm -Uvh mongodb-mms-9.0.1.532.20260928T1452Z.x86_64.rpm
```

Notes for this upgrade:

- With RPM packages, Ops Manager 5.0 and later do **not** overwrite your
  `conf-mms.properties` on upgrade — it keeps your current file. That is why your
  settings (central URL, Application Database connection string, email) survive the
  upgrade. You backed the file up in step 4 regardless.
- If a new Ops Manager version needs new properties in `conf-mms.properties`, you add
  them to the file after upgrading. The defaults are fine for this test setup.
- You do not re-run the install wizard. The upgraded Ops Manager reuses the existing
  configuration and Application Database.

### 8. Start Ops Manager and let it migrate

Start the service again:

```bash
sudo systemctl start mongodb-mms
```

The first 9.0 startup takes a few minutes: the JVM starts, Ops Manager runs its
pre-flight checks, and it migrates its schema in the Application Database, so port 8080
will not listen immediately. Watch progress:

```bash
sudo tail -f /opt/mongodb/mms/logs/mms0.log
```

It is normal for the log to pause at `Starting pre-flight checks` while the migration
runs. **Do not stop or restart the service because it looks stuck** — interrupting the
migration can leave Ops Manager in a bad state. Wait for it to finish and the port to come
up:

```bash
sudo ss -tlnp | grep 8080
```

If Ops Manager fails to start, check `mms0.log`. Repeated `Connection refused` to
`127.0.0.1:27017` means it cannot reach the Application Database — confirm the backing
`mongod` is still running (you left it up in step 5):

```bash
sudo systemctl status mongod --no-pager
```

### 9. Log in and confirm the new version

Open the console in a browser at the same URL as before:

```text
http://<OpsManagerHost>:8080
```

- `<OpsManagerHost>` — the hostname or IP of `opsmgr-1`.

Log in with your existing Global Owner account. A successful login against the upgraded
host is the primary signal the upgrade worked. Confirm the version in the console footer
reads 9.0.x, or on the host:

```bash
rpm -q mongodb-mms
```

### 10. Update the MongoDB Agents

After the upgrade, Ops Manager shows a banner such as **One or more agents are out of
date**. The Ops Manager Application is now 9.0, but the MongoDB Agents on your managed
nodes are still the version that shipped with 8.0.

1. In the console, click **Update Software Components** (or **Update Agents**) in the
   banner.
2. Review the proposed changes and confirm. Ops Manager rolls the matching agent version
   out to the managed nodes. If Ops Manager manages your MongoDB Tools, their versions are
   updated along with the agents.

This updates the **agents**, not your MongoDB server version — `new-rs0` and production
stay on MongoDB 8.0. Agent updates roll out without taking the databases down.

> **BI Connector note (9.0).** In Ops Manager 9.0 the BI Connector is disabled by default
> for new deployments, and it reached end of life on September 30, 2026. Existing BI
> Connector instances remain manageable, but updating the agents does **not** mean Ops
> Manager will provision or update a BI Connector for new deployments. This test setup
> does not use the BI Connector, so there is nothing to do here.

## Verification

- The console loads at `http://<OpsManagerHost>:8080` and you can log in with your
  existing account.
- The version now reads **9.0.x** (console footer or `rpm -q mongodb-mms`).
- The Ops Manager service is running:

  ```bash
  sudo systemctl status mongodb-mms --no-pager
  ```

- The Application Database `mongod` is still running on `127.0.0.1:27017`:

  ```bash
  sudo systemctl status mongod --no-pager
  ```

- If you did the optional step 2a, the Application Database now reports MongoDB **9.0.x**
  with FCV `9.0`:

  ```bash
  mongosh --quiet --port 27017 --eval 'db.version()'
  ```

  ```bash
  mongosh --quiet --port 27017 --eval 'db.adminCommand({getParameter:1, featureCompatibilityVersion:1})'
  ```

- After clicking **Update Software Components**, the "agents are out of date" banner
  clears and each managed project shows healthy, up-to-date agents.
- Your managed deployments (`new-rs0` and production) are still healthy and were not
  restarted or re-versioned by the upgrade. Confirm production directly:

  ```bash
  mongosh "mongodb://<prod-primary-fqdn>:27017/?replicaSet=<prod-rs-name>" -u <user> -p --quiet --eval 'rs.status().members.map(m => m.stateStr)'
  ```

  Expect one `PRIMARY` and healthy `SECONDARY` members.

## Troubleshooting

- **`rpm` reports "not an rpm package"**: the download was likely a small XML/HTML error
  page, not the package. Re-check with `file mongodb-mms-*.x86_64.rpm` and recopy the real
  rpm link from the release archive.
- **Console never comes up after start**: tail `/opt/mongodb/mms/logs/mms0.log`. If you
  see repeated `Connection refused` to `127.0.0.1:27017`, the Application Database is not
  reachable — confirm its `mongod` is running (`sudo systemctl status mongod --no-pager`)
  and that you did not change the connection string in `conf-mms.properties`.
- **Startup appears stuck at "Starting pre-flight checks"**: this is expected while the
  schema migration runs on first 9.0 startup. Do not restart the service — let it finish.
- **Login fails after upgrade / data looks unreadable**: this can happen if `gen.key`
  changed. Restore the backed-up key from step 4
  (`sudo cp -a ~/mms-gen.key.bak /opt/mongodb/mms/conf/gen.key`), then restart the
  service. `gen.key` must match the one used to encrypt the backing data.
- **Lost settings after upgrade** (central URL, email, connection string): restore your
  configuration from the backup
  (`sudo cp -a ~/mms-conf-mms.properties.bak /opt/mongodb/mms/conf/conf-mms.properties`)
  and restart. With RPM upgrades Ops Manager keeps your file, so this is rarely needed.
- **Pre-flight check error `Unrecognized VM option`**: a hand-edited config
  (`mms.conf`, `conf-mms.properties`, or the init script) contains an obsolete JVM flag
  such as `-XX:+UseParNewGC`. Remove that flag from the config file and start again.
- **Application Database won't start on 9.0 after the optional step 2a**: the usual cause
  is skipping the FCV bump — the 9.0 binary was started while FCV was still below 8.0.
  Reinstall the 8.0 binary, start `mongod`, set FCV to `8.0`
  (`setFeatureCompatibilityVersion:"8.0"`), confirm it reports `8.0`, then redo the binary
  swap to 9.0. Only raise FCV to `9.0` **after** the 9.0 binary is running healthy.
- **No supported downgrade**: Ops Manager has no supported downgrade path. If the upgrade
  fails, fix forward using your backups (and the Application Database `mongodump` if you
  took one), and contact MongoDB Support rather than trying to install the old package
  back over the new data.
