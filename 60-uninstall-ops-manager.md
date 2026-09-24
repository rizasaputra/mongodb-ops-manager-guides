# Uninstall Ops Manager (Keeping Production Safe)

This guide walks you through tearing down the test Ops Manager environment while keeping the existing production replica set (Project B) running uninterrupted.

## Overview

You will detach the production deployment from Ops Manager first, verify it keeps running on its own, and only then remove the test deployments, uninstall agents, and stop Ops Manager. The order matters: Ops Manager Automation manages the production processes, so tearing Ops Manager down carelessly could disrupt or stop production.

The key fact that makes this safe: **stopping Ops Manager from managing or monitoring a deployment does not touch its `mongod`/`mongos` processes.** They keep running uninterrupted unless you explicitly shut them down. So the plan is to cleanly release production, confirm it is healthy and self-standing, then tear down the rest.

> **Production-critical.** Project B is your real production replica set. Do this in an
> approved change window. The dangerous mistake is uninstalling the MongoDB Agent or
> shutting down Ops Manager *before* releasing Project B — an Agent that manages a
> process should not be removed while it still owns that process. Release first, verify,
> then remove.

This guide assumes the environment built across the earlier guides (Ops Manager on `opsmgr-1`, Project A `new-rs0`, Project B production). See the official docs for [Stop Managing and/or Monitoring One Deployment](https://www.mongodb.com/docs/ops-manager/current/tutorial/unmanage-deployment/) for reference.

## Prerequisites

- Access to the Ops Manager console with rights to modify and remove deployments.
- An approved change window for touching the production project (Project B).
- SSH access to all nodes (production and `new-rs0`) and to `opsmgr-1`, for removing agents afterward.
- Know your production replica set's connection details so you can verify it directly with `mongosh` after releasing it.

## Steps

Do Part A (release production) and confirm it before anything else. Parts B and C tear down the test pieces.

## Part A — Safely release the production deployment (Project B)

### 1. Terminate backups for Project B first

You must stop backups before you can remove a deployment from monitoring.

- If Project B uses the third-party backup integration, disable/terminate it from your Veritas/Cohesity console and remove the cluster's Third Party Managed status in Ops Manager.
- If Project B uses Ops Manager native backup, terminate the backup for the deployment in **Continuous Backup** so its snapshots and backup jobs stop.

### 2. Stop managing the production deployment

1. In the Ops Manager console, select Project B.
2. In the sidebar under the Database heading, click **Processes**.
3. For the production replica set, choose the option to stop managing it, and select **Completely remove from Ops Manager**. Since you are uninstalling Ops Manager entirely, there is no reason to keep monitoring — this option removes the deployment from both management and monitoring, and terminates its backups and deletes snapshots (you already terminated backups in step 1).

   (There is also an **Unmanage but continue to monitor** option, but that only makes sense if you intend to keep using Ops Manager for monitoring — not the case here.)
4. Click **Remove <Replica Set>**, enter a verification code if prompted, and confirm.
5. Review the proposed changes and click **Confirm & Deploy**.

After this, Ops Manager can no longer upgrade, stop, start, or reconfigure the production deployment — which is exactly what you want before removing its agents.

### 3. Verify production is still running on its own

Connect directly to the production replica set (not through Ops Manager) and confirm it is healthy:

```bash
mongosh "mongodb://<prod-primary-fqdn>:27017/?replicaSet=<prod-rs-name>" -u <user> -p
```

```javascript
rs.status()
```

Confirm one **PRIMARY** and healthy **SECONDARY** members, and that reads/writes work. Production is now independent of Ops Manager.

### 4. Return `mongod` to systemd control (do this before removing the Agent)

This is the critical step that keeps production alive long-term. When you attached
production to Automation, you disabled the `mongod` systemd service so systemd would not
race the Agent. Right now the Agent is what starts `mongod`. If you remove the Agent
without handing `mongod` back to systemd, nothing will restart `mongod` after a crash or
reboot — a latent outage.

So before removing the Agent, put `mongod` back under systemd on each production node:

1. Find the exact configuration the Agent uses to run `mongod` today, and copy it into
   the systemd config. This is the authoritative source — do not guess the settings.

   Find the running `mongod` and the config file it was started with:

   ```bash
   ps -ef | grep '[m]ongod'
   ```

   The command line shows `mongod -f <path>`. Under Automation this is usually
   `automation-mongod.conf` in the data directory, for example
   `/data/automation-mongod.conf`. View it:

   ```bash
   sudo cat /data/automation-mongod.conf
   ```

   Now create or update the systemd config at `/etc/mongod.conf` so it matches that file.
   Copy over at least these values exactly as the Agent has them:

   - `storage.dbPath` — the data directory (must be the same, or you lose the data).
   - `net.port` and `net.bindIp` — the same port and bind addresses the members use.
   - `replication.replSetName` — the exact replica set name.
   - `security.keyFile` — the internal-auth keyfile (you relocate this in the next step).
   - `systemLog.path` — where logs go.

   Confirm the systemd unit exists and starts `mongod` from `/etc/mongod.conf`:

   ```bash
   cat /usr/lib/systemd/system/mongod.service
   ```

   In the `[Service]` section, the `ExecStart` line should run `mongod` with
   `--config /etc/mongod.conf`. Since production runs the Enterprise server package, this
   unit is almost always already present — you are just verifying it.

   If the unit is genuinely missing, do **not** run a package install against the live
   node as a quick fix: reinstalling or upgrading `mongodb-enterprise-server` can trigger
   a service restart or change the binary version, disrupting the running `mongod`.
   Instead, either add a matching systemd unit file by hand, or handle the package
   install separately in a controlled maintenance window using the exact same server
   version already running. Then run:

   ```bash
   sudo systemctl daemon-reload
   ```
2. **Move the keyfile out of the Agent's directory.** Automation typically stores the
   internal-authentication keyfile under `/var/lib/mongodb-mms-automation/`. If
   `/etc/mongod.conf` points there, `mongod` will fail to start once you remove the
   Agent (step 5) and that directory is gone. Copy the keyfile to a permanent location
   owned by the `mongod` user and point `security.keyFile` in `/etc/mongod.conf` at it:

   ```bash
   sudo mkdir -p /etc/mongodb
   sudo cp /var/lib/mongodb-mms-automation/keyfile /etc/mongodb/keyfile
   sudo chown mongod:mongod /etc/mongodb/keyfile
   sudo chmod 400 /etc/mongodb/keyfile
   ```

   Then set `security.keyFile: /etc/mongodb/keyfile` in `/etc/mongod.conf`.
3. Enable the `mongod` service so it starts on boot:

   ```bash
   sudo systemctl enable mongod
   ```

4. Verify systemd can manage the process. The cleanest check is to let systemd own it:
   stop the Agent-started `mongod` and start it via systemd during your change window,
   confirming the member rejoins the replica set healthy. Do this one node at a time,
   keeping a primary available, so the replica set stays up.

Only proceed once every production node's `mongod` starts under systemd and the replica
set is healthy.

### 5. Remove the MongoDB Agent from the production nodes (optional)

Now that `mongod` is managed by systemd (step 4) and the deployment is released from
Ops Manager (step 2), removing the Agent is safe — `mongod` keeps running and will
restart on boot via systemd.

On each production node:

```bash
sudo systemctl disable --now mongodb-mms-automation-agent
```

```bash
sudo dnf remove -y mongodb-mms-automation-agent
```

Then clean up the Agent's leftover files, including the third-party backup oplog staging
directory you configured in the backup guide (`brs.thirdparty.baseOplogFilePath`, e.g.
`/var/lib/mongodb-mms/thirdparty-oplog`) and the Agent's own directory:

```bash
sudo rm -rf /var/lib/mongodb-mms/thirdparty-oplog
```

```bash
sudo rm -rf /var/lib/mongodb-mms-automation
```

Do not remove `/var/lib/mongodb-mms-automation` until you have copied out anything
`mongod` still needs (the keyfile — see step 4). If any config parameters you set via
Ops Manager are worth keeping, note that they now live in each node's `/etc/mongod.conf`
and stay in effect independently of Ops Manager.

## Part B — Tear down the test deployment (Project A)

`new-rs0` is a disposable test replica set, so you can be more aggressive here.

### 6. Shut down or remove `new-rs0`

If you want to keep the data, follow the same safe release as Part A (terminate backups, stop managing, verify). If the test replica set is truly disposable:

1. In Project A, terminate any backups for `new-rs0`.
2. On the **Processes** page, remove `new-rs0` and choose **Completely remove from Ops Manager**.
3. Confirm & Deploy.
4. Optionally shut down the `mongod` processes on the `new-1`, `new-2`, `new-3` nodes and remove the Agent from each:

   ```bash
   sudo systemctl disable --now mongodb-mms-automation-agent
   ```

## Part C — Stop and remove Ops Manager

Only after every deployment is released from Ops Manager.

### 7. Stop the Ops Manager service

On `opsmgr-1`:

```bash
sudo service mongodb-mms stop
```

### 8. Uninstall the Ops Manager package

```bash
sudo rpm -e mongodb-mms
```

### 9. Stop the Application Database and clean up (optional)

The Application Database holds only Ops Manager's own metadata, not your production data, so it is safe to remove once Ops Manager is gone.

1. Stop the Application Database `mongod` on `opsmgr-1`.
2. If you no longer need any of it, remove the data directories you created (for example `/data/appdb`, the backup head directory, and any file system snapshot store path).

### 10. Close firewall rules opened for Ops Manager (optional)

In `00-install-ops-manager.md` you opened ports for Ops Manager. Now that it is gone,
close what is no longer needed:

- Inbound `8080`/`8443` to `opsmgr-1` (the console).
- Any outbound rules that only existed for Ops Manager to reach the managed nodes or the backup target.

Leave the ports the production replica set needs for its own operation (for example
`27017` between production members) open.

## Verification

- **Production (most important)**: the production replica set responds directly via `mongosh` with a healthy `rs.status()` — one PRIMARY, healthy SECONDARY members — and serves reads and writes. It kept running throughout.
- Ops Manager no longer lists the production deployment.
- On each production node, `mongod` is enabled under systemd (`systemctl is-enabled mongod` returns `enabled`), its `security.keyFile` points to a permanent path outside the removed Agent directory, and the Agent and its leftover directories are gone.
- The Ops Manager web console is no longer reachable on `opsmgr-1` after you stop the service.
- If you removed them, the MongoDB Agent is gone from the nodes, and `mongod` on production still runs.

## Troubleshooting

- **"Remove" option unavailable**: you must terminate backups before removing a deployment from monitoring. Terminate backups first (step 1).
- **Worried about production during release**: releasing management does not stop `mongod` — the processes keep running. Verify directly with `rs.status()` (step 3) after removing the deployment. The release only removes Ops Manager's control, not your running database.
- **Accidentally shut a process while it was still managed**: if Automation still manages a deployment, it may try to restart processes to reach goal state. Release the deployment from management first; only then stop processes or remove agents.
- **Production went unreachable after removing an Agent**: this usually means `mongod` was not handed back to systemd first (step 4). The Agent was starting `mongod`, and once removed nothing restarts it. Restart `mongod` on the affected node using its config (`sudo systemctl start mongod`, after ensuring the systemd unit and `/etc/mongod.conf` are correct), and enable it for boot with `sudo systemctl enable mongod`.
- **`mongod` did not come back after a reboot**: the systemd service is likely still disabled from when Automation managed it. Run `sudo systemctl enable mongod` on each node (step 4) so it starts on boot.
