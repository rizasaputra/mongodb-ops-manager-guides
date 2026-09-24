# Back Up and Restore with a Dell Appliance (Third-Party Backup)

This guide walks you through integrating Ops Manager with your third-party backup platform (Veritas/Cohesity) to back up your production replica set to a Dell PowerProtect DD appliance, and then restore a snapshot into the `new-rs0` replica set.

## Overview

Your Dell PowerProtect DD appliance is integrated via Veritas/Cohesity, so both backup and restore use Ops Manager's **third-party backup integration** rather than Ops Manager's own (native) backup. In this model:

- The **third-party platform** (Veritas/Cohesity) drives and stores the backups on the Dell appliance, and drives restores.
- **Ops Manager** only exposes its Administration API so the platform can put a cluster into a backup-ready or restore-ready state, then return it to normal.

You configure the integration points in Ops Manager here; the actual backup, restore, and storage happen from the third-party console. This guide covers the full cycle: set up the integration once, back up the production replica set (Project B), then restore a snapshot into `new-rs0` (Project A).

> **Production note.** Project B is your real production system. The third-party platform
> performs the backup work, but activating backup and any initial copy add load. Do this
> during a low-traffic window.

> **Restore overwrites `new-rs0`.** Restoring replaces the data in `new-rs0` (Project A,
> from `10-provision-new.md`) with the production snapshot, so the target's existing
> contents are lost. That is expected here (production data landing in the test
> environment is intended), but be sure you are pointing at `new-rs0` and not any
> deployment whose data you want to keep.

This guide assumes you completed `25-monitoring-existing.md` (the production replica set is managed under Project B) and `10-provision-new.md` (the `new-rs0` replica set is running under Project A). See the official docs for [Backup and Restore with Third-Party Platforms](https://www.mongodb.com/docs/ops-manager/current/core/third-party-backup/) for reference, and your Veritas/Cohesity documentation for the vendor side.

## Important constraints

- **Ops Manager version**: third-party backup integration requires Ops Manager **8.0.8 or later**. Confirm your Ops Manager is at least this version.
- **One backup solution per cluster**: you cannot use both Ops Manager native backup and a third-party platform on the same cluster. (You can run different solutions on different clusters in the same project.)
- **Clocks synchronized**: all hosts must have synchronized clocks before configuring the integration (you set up `chronyd` on the nodes in `00-install-ops-manager.md`).
- MongoDB Support can help with the Ops Manager integration points, but backup/restore functionality and performance are handled by your third-party vendor.

## Prerequisites

- Ops Manager **8.0.8+** running on `opsmgr-1`.
- The production replica set managed by Ops Manager under Project B (from `25-monitoring-existing.md`).
- The `new-rs0` replica set running and managed by Ops Manager under Project A (from `10-provision-new.md`), reachable from Ops Manager on 27017 — this is the restore target.
- Admin access to the Ops Manager console (to create API keys and set config).
- A configured Veritas/Cohesity platform with access to the Dell PowerProtect DD appliance, able to reach Ops Manager's Administration API and the MongoDB nodes.
- If your third-party platform requires its own agent/client on backup or restore hosts, install it on every node of the clusters involved. (Cohesity/NetBackup, for example, requires its client on all nodes of the cluster used for restore.)

## Part A — Set up the third-party integration

Do this once. It applies to any project you back up or restore with the platform.

### 1. Generate an Ops Manager Administration API key

The third-party platform authenticates to Ops Manager with an API key. Create one with a backup-admin role. Check your Veritas/Cohesity docs for whether it needs global or project-level access.

For **global** access:

1. In the Ops Manager **Admin** console, click **General**, then **API Keys**.
2. Click **Create API Key**, add a description, and select **Global Backup Admin**.
3. Click **Next**, copy the **Public Key** and **Private Key**, and store them securely.
4. Click **Done**.

For **project-level** access (create one for each project you use — Project B for backup, Project A for restore):

1. In the project, expand **Access Manager** and select **Project Access**.
2. Click the **API Keys** tab, then **Create API Key**.
3. Add a description and select **Project Backup Admin**.
4. Click **Next**, copy the **Public Key** and **Private Key**, store them securely, and click **Done**.

### 2. Enable the third-party backup feature flag

1. In the Ops Manager **Admin** console, click **General**, then **Ops Manager Config**.
2. Click the **Custom** tab and add the key/value pair for your chosen access level:
   - Project level: key `mms.featureFlag.backup.thirdPartyManaged`, value `controlled`.
   - Global level: key `mms.featureFlag.backup.thirdPartyManaged`, value `enabled`.
3. Click **Save**.

### 3. Turn on third-party backup in project settings (project level only)

If you enabled the feature at the **project** level in step 2, enable it in each project you use (Project B and Project A):

1. In the project, click **Settings**.
2. Click the **Beta Features** tab and click **Backup Third Party Managed**.

### 4. Set the oplog output directory

This directory is a **staging area for oplog (transaction log) data**, not for full
backups. With backup enabled, the MongoDB Agent tails each replica set's oplog and
writes those entries here for the third-party platform to pick up. This is what enables
point-in-time recovery. The third-party platform stores the actual full snapshots on the
Dell appliance, not on your nodes.

Because only oplog entries pass through here (in small, rolling batches — native backup
uses roughly 10 MB compressed slices), this does **not** require storage anywhere near
your dataset size. You do not need to double your node storage. Size it to hold the
oplog that accumulates between the platform consuming it; the exact amount depends on
your write rate.

1. In the Ops Manager **Admin** console, click **General**, then **Ops Manager Config**.
2. On the **Custom** tab, add the key `brs.thirdparty.baseOplogFilePath` with a value
   that is a directory the MongoDB Agent can read and write, for example
   `/var/lib/mongodb-mms/thirdparty-oplog`.
3. Click **Save**.
4. On each node, create the directory if needed and verify the MongoDB Agent user can
   read and write to it.

## Part B — Back up the production system

### 5. Ensure MongoDB Agents are installed on the production nodes

Your production nodes already run the MongoDB Agent from `25-monitoring-existing.md`. If any server you want to back up does not have it:

1. In Project B, click **Deployment**, the **Agents** tab, then **Downloads & Settings**.
2. Select the host OS and follow the instructions to install the Agent on each server.

### 6. Activate backup and set the production cluster to Third Party Managed

1. In Project B, click **Deployment**, then the **Servers** tab.
2. For each server, click the menu next to its MongoDB Agent and click **Activate Monitoring** and **Activate Backup**.
3. Click **Review & Deploy**, review the changes, and click **Confirm & Deploy**.
4. Click **Continuous Backup** in the sidebar.
5. Hover over the **Status** column for the production replica set and click **Manage**, then **Manage** again in the modal.

The cluster's Continuous Backup status changes to **Third Party Managed**.

### 7. Run the backup from the Veritas/Cohesity console

The Ops Manager side is now ready. Switch to your Veritas/Cohesity console: register the Ops Manager Administration API key, point the platform at the production cluster, and configure the backup schedule and retention that store snapshots on the Dell PowerProtect DD appliance. Follow your vendor's documentation.

From here, the platform calls the Ops Manager API to put the cluster into a backup-ready state, performs the backup to the Dell appliance, and returns the cluster to normal.

## Part C — Restore into the new environment

Restore the production snapshot from the Dell appliance into `new-rs0` (Project A).

### 8. Prepare the restore target

1. Confirm `new-rs0` is healthy in Project A: on the **Deployment** page, all three members are at goal state with one **PRIMARY** and two **SECONDARY**.
2. Confirm `new-rs0` is managed by Automation (it is, from `10-provision-new.md`) so Ops Manager can move it into a restore-ready state on the platform's request.
3. Make sure the third-party integration is set up for Project A too — the feature flag (Part A, steps 2-3) and, if using project-level access, an API key for Project A (Part A, step 1).
4. Accept that the restore overwrites `new-rs0`. If anything on it matters, capture it first.

### 9. Run the restore from the Veritas/Cohesity console

Switch to your Veritas/Cohesity console to drive the restore. Following your vendor's documentation:

1. Register or confirm the Ops Manager Administration API key and connection.
2. Select the production backup snapshot on the Dell appliance to restore (choose the point in time if the platform supports point-in-time recovery).
3. Choose `new-rs0` (in Project A) as the **restore target**.
4. Start the restore.

The platform calls the Ops Manager API to put `new-rs0` into a restore-ready state, writes the snapshot data to the target, then calls the API again to return `new-rs0` to a normal running state.

## Verification

Backup:

- In Project B, the production replica set's **Continuous Backup** status shows **Third Party Managed**.
- A backup job completes from the third-party console, and its snapshots are stored on the Dell PowerProtect DD appliance (verify in the vendor console / appliance).
- No production data was modified — the platform coordinates read-oriented backups through Ops Manager.

Restore:

- In Project A, `new-rs0` is back at **goal state reached**, with one **PRIMARY** and two **SECONDARY** members.
- The restored production data is present. Connect to the primary and spot-check:

  ```bash
  mongosh "mongodb://new-1.example.dev:27017/?replicaSet=new-rs0"
  ```

  ```javascript
  show dbs
  ```

  ```javascript
  // pick a known production database and collection, then confirm it has data
  db.getSiblingDB("<known-db>").<known-collection>.countDocuments()
  ```

  The databases, collections, and document counts should match what you expect from the production snapshot.
- Your Veritas/Cohesity console reports both the backup and restore jobs as successful.

## Troubleshooting

- **Third-party option not available**: confirm Ops Manager is **8.0.8+**, the feature flag from Part A step 2 is set, and (for project-level) the Beta Feature in Part A step 3 is on for the relevant project.
- **Cluster cannot become Third Party Managed**: make sure that cluster is not already using Ops Manager native backup. Only one backup solution per cluster is allowed.
- **Agent cannot write oplog**: verify the `brs.thirdparty.baseOplogFilePath` directory exists and the MongoDB Agent user has read/write permission on every node.
- **Restore target not selectable in the vendor console**: confirm `new-rs0` is managed by Ops Manager (Automation), the third-party feature is enabled for Project A, and (if required) the vendor's agent/client is installed on all `new-rs0` nodes.
- **`new-rs0` stuck in restore-ready / not returning to normal**: check the third-party job status; the platform is responsible for calling the API to return the cluster to normal. Review the vendor console and Ops Manager's deployment status.
- **Vendor console cannot reach Ops Manager**: confirm the API key role (Global or Project Backup Admin) matches what your vendor requires, and that the platform can reach the Ops Manager Administration API.
- **Backup/restore functionality or performance issues**: these are handled by the third-party platform — contact your Veritas/Cohesity vendor. MongoDB Support covers the Ops Manager integration points only.
