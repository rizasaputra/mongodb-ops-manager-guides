# Native Ops Manager Backup and Restore (No Third-Party Integration)

This guide walks you through backing up and restoring a replica set using Ops Manager's own native backup, storing snapshots on a local file system with no third-party platform involved.

## Overview

Unlike the third-party integration in `45-backup-restore-third-party.md`, here **Ops Manager itself** takes, stores, and restores the snapshots. You point Ops Manager at storage it manages, enable backup on a deployment, then restore a snapshot back with Ops Manager driving the whole process.

This guide targets the `new-rs0` replica set in Project A (from `10-provision-new.md`) — a test/lab deployment, so it is safe to exercise native backup and restore against it.

### How native backup stores data

Native backup uses two kinds of storage, and you choose the type of each.

**Snapshot store** — holds the full point-in-time snapshots of your data. Options:

- **File system store** — snapshots stored as files on a local or network-attached
  file system path. Simplest to set up; no extra database or object storage needed.
- **Blockstore** — snapshots stored in a dedicated MongoDB database, compressed and
  de-duplicated at the block level. Efficient for large data, but needs a dedicated
  MongoDB deployment to host it.
- **S3-compatible store** — snapshots stored as objects in an S3 or S3-compatible
  bucket. Good when you already have object storage; needs the bucket and credentials.

**Oplog store** — holds the ongoing stream of oplog entries so Ops Manager can offer
point-in-time restores (not just whole snapshots). Options:

- **Local oplog store** — oplog slices stored in a MongoDB database you run.
- **S3-compatible oplog store** — oplog slices stored in an S3-compatible bucket.

Every Ops Manager deployment needs at least one oplog store, regardless of the snapshot
store type.

**This guide uses the simplest combination**: a **file system snapshot store** (a local
path on `opsmgr-1`) and a **local oplog store** (a MongoDB instance you point Ops
Manager at). No object storage or dedicated blockstore cluster required.

> **One backup solution per cluster.** A cluster cannot use both native backup and a
> third-party platform at the same time. Use native backup here on `new-rs0`, which is
> not backed up by the third-party integration.

This guide assumes you completed `10-provision-new.md` (the `new-rs0` replica set is running and managed under Project A). You enable the Backup Daemon as part of these steps. See the official docs for [Manage File System Snapshot Storage](https://www.mongodb.com/docs/ops-manager/current/tutorial/manage-filestore-storage/) and [Restore a Replica Set from a Snapshot](https://www.mongodb.com/docs/ops-manager/current/tutorial/restore-replica-set/) for reference.

## Prerequisites

- Ops Manager running on `opsmgr-1` with the Backup Daemon binary present (it ships with Ops Manager; you enable it in step 2).
- A **Global Owner** account, needed to reach the Admin area.
- The `new-rs0` replica set running and managed by Ops Manager (Automation) under Project A (from `10-provision-new.md`).
- On `opsmgr-1`, enough free disk for the head directory and the snapshots, and paths you will create and own as `mongodb-mms:mongodb-mms` (this guide uses `/data/backup/head` and `/data/backup/fs-store`).
- The `mongod` binary available on `opsmgr-1` so you can run a small dedicated `mongod` for the oplog store (created in step 4). Do not reuse the Ops Manager Application Database for this.

## Steps

### 1. Create the backup directories on `opsmgr-1`

You need two directories on `opsmgr-1`, both created up front:

- The **head directory** — a working directory where the Backup Daemon assembles and
  serves the head database (its working copy of the backed-up data) and queryable
  restores.
- The **file system store directory** — where the snapshots themselves are stored (used in
  step 3).

Both must exist and be owned by the user Ops Manager runs as before you point the console
at them, or you get `Unable to collect stats from the configured directory`. Package
installs on Linux run Ops Manager and the Backup Daemon as the **`mongodb-mms`** user and
group. SSH to `opsmgr-1` and create both:

```bash
sudo mkdir -p /data/backup/head /data/backup/fs-store
```

```bash
sudo chown -R mongodb-mms:mongodb-mms /data/backup/head /data/backup/fs-store
```

```bash
sudo chmod 755 /data/backup/head /data/backup/fs-store
```

This is on **`opsmgr-1`**, and the owner is **`mongodb-mms`** — not `mongod`. (The `/data`
directory on the MongoDB nodes is owned by `mongod`; do not confuse the two.) If you
installed Ops Manager under a different user, chown to that user instead — check with
`ps -o user= -p $(pgrep -f 'mms' | head -1)`.

### 2. Enable the Backup Daemon and set the head directory

On a fresh Ops Manager, the first time you open backup administration you land on the
**Backup Initial Configuration** wizard, which walks you through the head directory, the
daemon, and the snapshot store in order.

1. Open the **Admin** area. The **Admin** link is in the **top-right corner** of the Ops
   Manager Application (not in the left project sidebar). It only appears if your account
   has the **Global Owner** role — if you do not see it, log in as a Global Owner or have
   one grant you the role. Click the **Backup** tab.
2. On the **Backup Initial Configuration** page, find the **HEAD directory** field. Enter
   the **plain path** you created:

   ```text
   /data/backup/head
   ```

   Enter the path only — no `<hostname>:` prefix and no trailing slash (Ops Manager adds
   the slash for you). Click **Update**. The page re-checks the directory; the
   "Unable to collect stats" message clears once the daemon can read it.
3. Click **Enable Daemon** to turn on the Backup Daemon with that head directory.

Changes here can take up to 5 minutes to take effect.

### 3. Configure a file system snapshot store

After enabling the daemon, the wizard offers three snapshot store types: **Configure a
File System Store**, **Configure a Blockstore**, and **Configure an S3 Blockstore**.
Choose **Configure a File System Store** — this guide stores snapshots on a local file
system, with no blockstore cluster or S3 bucket.

At minimum you only need the **Path** to the store directory you created in step 1:

```text
/data/backup/fs-store
```

If the form only shows a Path field, enter that and Save — Ops Manager fills in a name and
defaults. If you (or the wizard) switch to the **Advanced** view, you cannot switch back to
the simple form, and you must then fill in these fields:

- **File System Store Name** — any label, e.g. `fs-store`.
- **Path** — `/data/backup/fs-store` (from step 1).
- **MMapV1 Compression Setting** — defaults to **Gzip**. Set it to **None**.
- **WiredTiger Compression Setting** — defaults to **None**. Leave it **None**.
- **New Assignment Enabled** — leave selected so backup jobs can be assigned to this store.

Save.

**About the two compression settings and queryable backups.** For queryable backups, Ops
Manager requires uncompressed snapshots, and the two settings cover the two storage
engines separately — so the documented rule is to set **both** to **None**. In practice
only one applies to you: `new-rs0` runs **WiredTiger** (MMAPv1 was removed from MongoDB
years ago), so the **WiredTiger** setting is the one that governs your snapshots, and it is
already **None** by default. The **MMapV1** setting defaults to **Gzip** but is irrelevant
to your WiredTiger data; set it to **None** anyway to match the rule and avoid confusion.
Also note that for MongoDB FCV 6.0+ (your 8.0 set qualifies) Ops Manager does not compress
file system snapshots at all, so queryable backup works regardless.

### 4. Create the oplog store

The oplog store is a MongoDB database where Ops Manager keeps the oplog slices that make
point-in-time restores possible. **It is required** — Ops Manager needs at least one oplog
store before it will let you enable backup for any deployment (it is not just a
point-in-time add-on you can skip).

You need a dedicated `mongod` for it. **Do not reuse the Ops Manager Application Database**
`mongod` (the one on `opsmgr-1:27017`): the docs advise against putting the oplog store in
the Application Database, and that `mongod` binds to `127.0.0.1` only, so Ops Manager
cannot reach it by hostname anyway. Instead run a small separate `mongod` on a different
port.

1. **On `opsmgr-1`, start a dedicated `mongod` for the oplog store.** Use a different port
   (`27018`) and a data path separate from the snapshot store. Bind it to both `localhost`
   and the host's own address so the Backup Daemon can connect:

   ```bash
   sudo mkdir -p /data/backup/oplog-store
   ```

   ```bash
   sudo chown -R mongodb-mms:mongodb-mms /data/backup/oplog-store
   ```

   ```bash
   sudo -u mongodb-mms mongod --port 27018 --dbpath /data/backup/oplog-store --bind_ip localhost,<opsmgr-fqdn> --fork --logpath /data/backup/oplog-store/mongod.log
   ```

   Replace `<opsmgr-fqdn>` with the output of `hostname -f` on `opsmgr-1`. Confirm it is
   up:

   ```bash
   mongosh "mongodb://localhost:27018/?directConnection=true" --eval "db.runCommand({ ping: 1 })"
   ```

   (This `mongod` is started by hand, so it will not survive a reboot — fine for this test.
   For a longer-lived setup, run it as a systemd service.)

2. In the console, click the **Admin** link, then the **Backup** tab, then the **Oplog
   Storage** page.
3. Add an oplog store and set:
   - **Name** — a label, e.g. `oplog-store`.
   - **Datastore Type** — **Standalone** (this is a single `mongod`).
   - **MongoDB Hostname** — `localhost` (the Backup Daemon runs on the same host). You can
     also use the `opsmgr-1` FQDN, since the `mongod` is bound to it.
   - **MongoDB Port** — `27018`.
   - **Username** / **Password** — leave blank; this `mongod` has no auth.
   - **New Assignment Enabled** — leave selected so backup jobs can use this oplog store.
4. Save.

### 5. Activate the Backup function on one server

Backup, like Monitoring, is an **agent function** that must be turned on before you can
enable backup for a deployment. Do this first — otherwise the **Start Backup** button in
the next step stays greyed out.

Unlike Monitoring (which you activated on all three nodes for failover in
`20-monitoring-new-and-alerting.md`), **you only need the Backup function on ONE server**
in the project. A single active Backup agent handles backing up every deployment in the
project.

**Which host, and primary vs. secondary?** It does not matter. Activating the Backup
*function* just chooses which host runs the Backup agent — a client process. It is
independent of which replica set member the backup reads from: Ops Manager's Backup agent
polls the replica set's **primary** and streams its oplog, and it follows the primary
automatically across elections. So pick any `new-rs0` host; whether that host currently
holds a primary or secondary is irrelevant.

1. Select Project A, go to **Deployment > Servers**.
2. On **one** of your `new-rs0` hosts, click the actions menu (**⋯** / gear) and click
   **Activate Backup**. (Monitoring should already show active on it.)
3. A banner appears. Click **Review & Deploy**, then **Confirm & Deploy**.
4. Wait a minute or two for the deploy to finish. The host should then show
   **Backup - active** on the Servers tab.

> **Single point of failure.** With Backup active on only one host, if that host goes
> down there is no other Backup agent to take over, so backups **pause** until it returns
> (or until you activate Backup on another server); already-stored snapshots are not lost,
> and Ops Manager may raise a "Backup requires resync" alert if it falls far behind. For a
> lab this is fine. For high availability, activate the Backup function on 2+ hosts — like
> Monitoring, Ops Manager keeps the extras on standby and hands the work off if the active
> Backup agent's host fails. This is separate from replica set failover, which the backup
> already tolerates by following the new primary.

### 6. Enable backup for `new-rs0`

With the Backup function active, return to your project and click **Continuous Backup** in
the sidebar. Because Backup is already activated, there is **no "Begin Setup" wizard** —
you land directly on the **Overview** table listing `new-rs0`, with columns for Status,
Last Snapshot, Next Snapshot, and Last Oplog Slice.

On the `new-rs0` row, click **START** in the **Status** column.

Once started, Ops Manager performs an **initial sync** to seed the backup, then keeps it
current from the oplog. The Status column updates and the snapshot columns begin to fill
in.

(If you had **not** activated the Backup function first, this page would instead show a
**Begin Setup** button whose wizard just sends you to activate Backup on a server — which
is exactly what step 5 already did, so you skip it entirely.)

### 7. Take and confirm a snapshot

A few minutes after you click START, the **Status** column changes to **active**. Note
that **active means backup is running (tailing the oplog) — it does not by itself mean a
snapshot exists yet.** You can only restore once at least one snapshot has been taken.

1. Confirm a snapshot exists before moving on. On the `new-rs0` row, the **Last Snapshot**
   column should show a timestamp (not empty). You can also open the row's ellipsis
   (**⋯**) and click **View All Snapshots** — there should be at least one entry.
2. The first snapshot is taken **on the schedule**, which defaults to **every 6 hours** —
   so on a fresh backup you might wait hours for it. For a lab, do not wait: open the row
   ellipsis → **Edit Snapshot Schedule** and lower "take snapshots every … hours" to the
   smallest value allowed (for example 1 hour), then save so the first snapshot lands
   sooner. (Once it fires, a small dataset snapshots quickly — expect only a few minutes,
   e.g. ~4 minutes for ~500 MB.) Some Ops Manager builds also expose a **Take Snapshot
   Now** option once the first scheduled snapshot exists, but do not count on it — if you
   do not see it, just rely on the schedule.
3. The row's ellipsis also has **Select Preferred Member to Backup** (which member the
   agent reads from; leave it default so it follows the primary). Not required for this
   test.

### 8. Restore a snapshot (automated restore)

Once at least one snapshot exists (from step 7), you can restore. Use the automated
restore so Ops Manager does the work. Automation removes all existing data from the target
cluster and replaces it with the snapshot.

1. **Start the restore.** In Project A, go to **Continuous Backup**. The **Restore
   History** tab is empty on a first run and has no restore button, so do not start from
   there — instead start the restore from the `new-rs0` row's **ellipsis (⋯)**, or open
   the ellipsis → **View All Snapshots** and restore a listed snapshot from that page.
2. **Choose the restore point.** You may be offered up to three options — **Snapshot** (a
   specific stored snapshot), **Point In Time**, and **Oplog Timestamp**. In a fresh lab
   with little write activity, Point In Time / Oplog Timestamp may not be selectable yet
   (they need enough continuous oplog history to define a restorable range). That is fine —
   just restore the **Snapshot** you have.
3. Click **Next**, then **Choose Cluster to Restore to**.
4. Select **Project** (Project A) and **Cluster to Restore to** (`new-rs0`). The target
   must be managed by Ops Manager.
5. Click **Restore**. Ops Manager opens a confirmation window; click **Confirm & Deploy**
   to apply it. Automation then wipes `new-rs0` and rebuilds it from the snapshot, the same
   staged deploy model as other changes.

## Verification

- On the **Snapshot Storage** page, `fs-store` is listed and snapshot files appear under its path on `opsmgr-1`.
- On Project A's backup page, `new-rs0` shows backup **active** with the initial sync **completed** and at least one **snapshot** with a recent timestamp.
- After a restore, `new-rs0` returns to **goal state reached** with one **PRIMARY** and two **SECONDARY**, and the data matches the restored snapshot. Connect and spot-check:

  ```bash
  mongosh "mongodb://new-1.example.dev:27017/?replicaSet=new-rs0"
  ```

  ```javascript
  show dbs
  ```

## Troubleshooting

- **"Unable to collect stats from the configured directory"**: two common causes. First, the value in the **HEAD directory** field must be a **plain path** (`/data/backup/head`) — not `<hostname>:/path/`. The `<hostname>:/path/` format belongs to the separate **Pre-Configure New Daemon** box on the standing Daemons page, not the Backup Initial Configuration wizard; entering it here makes the daemon check an invalid path. Second, the directory must exist and be owned by the Ops Manager user: `sudo mkdir -p /data/backup/head && sudo chown -R mongodb-mms:mongodb-mms /data/backup/head` (confirm with `ls -ld /data/backup/head`). After correcting either, click **Update** again; it can take up to ~5 minutes to re-check.
- **Oplog store host unreachable / `ECONNREFUSED` on the oplog `mongod`**: the oplog `mongod` is likely bound to `127.0.0.1` only (the Application Database default), so it refuses connections by hostname/IP. Do not use the Application Database (`opsmgr-1:27017`) as the oplog store. Use the dedicated `mongod` from step 4, started with `--bind_ip localhost,<opsmgr-fqdn>`, and register it as `localhost:27018`. Verify with `mongosh "mongodb://localhost:27018/?directConnection=true" --eval "db.runCommand({ ping: 1 })"`.
- **No snapshots appear**: confirm the Backup Daemon is running, the oplog store exists, and the file system store has **New Assignment Enabled** selected.
- **Access errors writing snapshots**: confirm the snapshot store path is owned by `mongodb-mms:mongodb-mms` and has enough free space.
- **Point-in-time restore time not available**: an oplog gap exists for that period. Choose a time within the restorable ranges the restore dialog shows. The oplog store keeps 24 hours by default.
- **Restore target not selectable**: the target cluster must be managed by Ops Manager (Automation). Confirm `new-rs0` is healthy and managed.
- **Queryable backup cannot query a snapshot**: it is compressed. Set the store's compression to off, and note Ops Manager does not compress file system snapshots for MongoDB FCV 6.0+ anyway.
