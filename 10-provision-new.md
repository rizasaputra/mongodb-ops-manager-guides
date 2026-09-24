# Provision a New MongoDB Replica Set with Automation

This guide walks you through using Ops Manager automation to deploy a brand-new MongoDB replica set onto three empty nodes.

## Overview

You will create a project in Ops Manager (Project A — New), install the MongoDB Agent on three empty nodes, and then use the Ops Manager web console to deploy a three-member MongoDB replica set onto those nodes.

With Ops Manager automation you do not install MongoDB by hand. You install only the MongoDB Agent on each node; Ops Manager then pushes the MongoDB Enterprise binaries and the replica set configuration to the nodes. This is the new-deployment workflow, distinct from attaching an already-running deployment.

This guide assumes you completed `00-install-ops-manager.md` and have a running Ops Manager on `opsmgr-1`. See the official docs for [Deploy a Replica Set](https://www.mongodb.com/docs/ops-manager/current/tutorial/deploy-replica-set/) and [MongoDB Agent Prerequisites](https://www.mongodb.com/docs/ops-manager/current/tutorial/install-mongodb-agent-prereq/) for reference.

## Prerequisites

- A running Ops Manager on `opsmgr-1` and a Global Owner account (from `00-install-ops-manager.md`).
- Three empty nodes `new-1`, `new-2`, `new-3`, each:
  - **OS**: RHEL 9.4 (64-bit).
  - **Resources**: at least 2 CPU cores and 2 GB RAM (MongoDB Agent minimum). Provision more for a realistic replica set.
  - **No MongoDB installed** — the nodes are empty. Ops Manager installs MongoDB for you.
  - **Time synchronized.** On RHEL 9, enable time sync with `chronyd` so replica set
    elections and oplog timestamps stay consistent:

    ```bash
    sudo dnf install -y chrony
    ```

    ```bash
    sudo systemctl enable --now chronyd
    ```

    Confirm with `timedatectl` that the system clock is synchronized.
- Network access:
  - Each new node can reach `opsmgr-1` on 8080/8443.
  - `opsmgr-1` can reach each new node, and the new nodes can reach **each other** on 27017.
- Each node self-identifies by its FQDN. Confirm on each node:

  ```bash
  hostname -f
  ```

  The result should be a unique FQDN such as `new-1.example.dev`, resolvable from the other hosts (via DNS or `/etc/hosts`).

## Steps

Steps 1-3 are done in the Ops Manager web console. Step 4 is run on each new node. Steps 5-6 return to the console.

### 1. Create the project

1. In the Ops Manager console, open the **Organizations** menu and select your organization.
2. Click **New Project** and give it a name for this deployment, for example `Project A - New`, then click **Next**.

   Keeping this deployment in its own project isolates it from the production system you attach in a later guide.
3. On the **Select A Default Server Type** screen, choose the server type Ops Manager should assume for this project. For this test deployment, the default is fine — select it and click **Continue**. (This only sets a default and does not restrict what you can deploy.)
4. On the **Add Members and Set Permissions** screen, optionally add other Ops Manager users to the project and assign their roles. For this test setup you can leave it empty and continue as the sole owner — you can add members later from **Project Settings**. Click through to finish creating the project.

### 2. Generate an Agent API Key

The MongoDB Agent needs one Agent API Key per project to talk to Ops Manager.

1. With Project A selected, click **Deployment**.
2. Go to **Agents**, then **Agent API Keys**.
3. Click **Generate**, enter a description that identifies the key's purpose (for example `Project A automation agent`), and click **Generate** again.
4. Copy the key immediately and store it securely. Ops Manager shows the full key only once. Treat it like a password.

### 3. Open the Agent install instructions for your OS

Ops Manager generates copy-paste install commands tailored to your project (they already
include your Project ID, Agent API Key, and Ops Manager URL). Bring them up in the
console:

1. With Project A selected, click **Deployment** in the sidebar, then **Agents**.
2. Click **Downloads & Settings**.
3. Under **Select Your Server's Operating System**, choose your platform — for these
   nodes, **RHEL 9 / CentOS 9** (RPM, x86_64).
4. A pop-up window opens with the **Install Agent Instructions** for that OS. Keep this
   window open; you will run its commands on each node in the next step.

The instructions list the exact Agent version and download URL for your Ops Manager, so
you do not have to guess them.

### 4. Run the Install Agent Instructions on each new node

Run the commands from the **Install Agent Instructions** pop-up on `new-1`, `new-2`, and
`new-3`. Log in to each node as `root` or a user with `sudo`, then follow the pop-up in
order. For the RHEL 9 RPM path the instructions walk you through:

- Downloading and installing the MongoDB Agent `.rpm`.
- Writing your Project ID and Agent API Key into
  `/etc/mongodb-mms/automation-agent.config` (the commands come pre-filled with your
  values).
- Creating the data directory the Agent owns and starting the Agent, for example:

  ```bash
  sudo mkdir -p /data
  ```

  ```bash
  sudo chown `whoami` /data
  ```
- Installing required 3rd-party dependencies as listed in the [documentation](https://www.mongodb.com/docs/ops-manager/current/tutorial/provisioning-prep/?os-enterprise-deps=rhel&rhel-enterprise-version=rhel-8). 

Copy each command from the pop-up exactly as shown rather than retyping the URLs or keys.
Repeat on all three new nodes. Within a minute or two, each node appears on the Ops
Manager **Deployment** page under Project A.

> **Closing the SSH session is safe.** The pop-up starts the Agent with `nohup ... &`,
> which detaches it from your shell and keeps it running after you log out. So after the
> Agent starts you can press Enter to get your prompt back (you do not need to `Ctrl+C`,
> and `Ctrl+C` will not stop the backgrounded Agent anyway), then close the SSH session
> and move on to the next node. Before you leave a node, confirm the Agent is running:
>
> ```bash
> pgrep -af mongodb-mms-automation-agent
> ```
>
> You should see the Agent process with a PID. The node also appearing on the
> **Deployment** page confirms it reported in successfully.
>
> Note: the `nohup` command from the pop-up is a run-once foreground start, so the Agent
> will not come back automatically after a reboot. That is fine for this test walkthrough.
> For a longer-lived setup you would run the Agent as a systemd service
> (`sudo systemctl enable --now mongodb-mms-automation-agent`) so it starts on boot.

### 5. Deploy the replica set from the console

Back in the Ops Manager console, with Project A selected:

1. Click **Deployment** in the sidebar.
2. Click the **Add** arrow in the top-right and select **New Replica Set**.
3. In **Replica Set Configuration**, set:
   - **Replica Set Id** — a unique name, e.g. `new-rs0`. You cannot change this later, and names must be unique within the project.
   - **Version** — a MongoDB 8.0 Enterprise build.
   - **Data Directory** — e.g. `/data` (the directory the Agent owns from step 4).
   - **Log File** — e.g. `/data/mongodb.log`.
4. Under **Member Configuration**, you configure three members. Each member has a
   **Hostname** dropdown that lists the hosts running the Agent under this project. Set
   each of the three members to a **different** node — member 1 to `new-1`, member 2 to
   `new-2`, member 3 to `new-3` — so every replica set member lands on its own host. If
   two members point at the same hostname you would be stacking members on one node,
   which is not what you want. Leave **Port** at `27017` since each node runs a single
   mongod, and keep all three as **Default** (data-bearing, votable) members.
5. Review the replication and write-concern defaults. The defaults are fine for a test replica set.
6. Save the configuration. Ops Manager stages your changes but does **not** deploy them
   until you explicitly review and confirm:
   - Click **Review & Deploy** at the top of the page. Ops Manager shows a summary of the
     pending changes (the new replica set and its members).
   - In the pop-up, click **Confirm & Deploy** to start the deployment. (If you instead
     see options to cancel or keep editing, the deployment has not started yet.)

   Until you click **Confirm & Deploy**, nothing happens on the nodes — the plan just
   sits there staged, which can look like the deploy is stuck. Watch progress on the
   **Deployment** view once you confirm.

   The first deploy takes a few minutes — Ops Manager downloads the MongoDB binaries to
   each node, starts `mongod`, and initiates the replica set. Expect roughly 3-10 minutes
   depending on network speed and node resources (the initial binary download is usually
   the slowest part).

   > **Green status also depends on Monitoring.** Automation only deploys and runs the
   > `mongod` processes. The green status dots (and Ping, metrics, and "Data Available")
   > on the Deployment/Processes view come from the Agent's **Monitoring** function, which
   > is a separate thing that may not be on yet. If your replica set is actually running
   > (you can confirm with `rs.status()` — see Verification) but the processes stay grey
   > with empty Ping and a "No Monitoring detected" banner, the deploy succeeded and you
   > just need to enable Monitoring. Enabling Monitoring on each node is the first step of
   > the monitoring guide (`20-monitoring-new-and-alerting.md`); follow it and the dots
   > turn green once metrics start flowing.

### 6. (Optional) Seed data from another database with mongodump/mongorestore

The new replica set starts empty. If you want realistic data in it — for example so the
Performance Advisor has something to analyze later, or to test backups against real
content — copy a database from another MongoDB deployment using `mongodump` (to export)
and `mongorestore` (to import). These tools ship with the MongoDB Database Tools.

1. Dump a database from the source deployment. This writes BSON files to a `dump/`
   directory:

   ```bash
   mongodump --uri="mongodb://<source-host>:27017/<source-db>" --out=dump
   ```

   - `<source-host>` — the source MongoDB host (add credentials/`?authSource=` if it uses auth).
   - `<source-db>` — the database to copy. Omit `/<source-db>` to dump all databases.

2. Restore the dump into `new-rs0`. Point the URI at the replica set so the write goes to
   the primary:

   ```bash
   mongorestore --uri="mongodb://new-1.example.dev:27017,new-2.example.dev:27017,new-3.example.dev:27017/?replicaSet=new-rs0" dump
   ```

   - Add `--drop` if you want `mongorestore` to drop each collection before restoring it,
     so repeated runs stay clean.
   - Restore a single database with `--nsInclude="<db>.*"`, or remap names with
     `--nsFrom`/`--nsTo`.

3. Confirm the data landed:

   ```bash
   mongosh "mongodb://new-1.example.dev:27017/?replicaSet=new-rs0" --eval "show dbs"
   ```

Only copy data you are allowed to place in this test environment. If the source is a
real system, treat its data accordingly.

## Verification

- Go to **Deployment > Processes**. This tab lists each `mongod` process in `new-rs0`
  with a status indicator next to it. Once the deploy finishes **and Monitoring is
  enabled**, every process turns **green** (goal state reached / running) and shows Ping
  and metrics.
- The same Processes list labels one member **PRIMARY** and the other two **SECONDARY**.
  You do not need to click into anything to see this — the color and role show in the
  list itself. Clicking the replica set name (`new-rs0`) or an individual process name
  just drills into that item's details.
- **If the processes stay grey with empty Ping and "No Data Available"**, the deployment
  is not broken — the Agent's Monitoring function is just not enabled yet. Confirm the
  replica set is genuinely healthy with the `rs.status()` check below, then enable
  Monitoring by following the first step of `20-monitoring-new-and-alerting.md`. The dots
  go green once monitoring data starts arriving.
- Optionally, confirm the replica set status with `mongosh`. Ops Manager does not add
  `mongosh` to your `PATH`, so on a node you either install it or use the copy Ops
  Manager already bundled under the automation directory. Find the bundled one with:

  ```bash
  find /var/lib/mongodb-mms-automation -name mongosh -type f 2>/dev/null
  ```

  Then run `rs.status()` against the local `mongod` (adjust the path to the version you
  found):

  ```bash
  /var/lib/mongodb-mms-automation/mongosh-linux-x86_64-<version>/bin/mongosh --host localhost:27017 --eval "rs.status().members.map(m => ({name: m.name, state: m.stateStr, health: m.health}))"
  ```

  You should see three members with `health: 1`, one in state `PRIMARY` and two in
  `SECONDARY`. That confirms a healthy replica set regardless of what the console shows.

  If you prefer a real `mongosh` on the node, install the package instead:

  ```bash
  sudo tee /etc/yum.repos.d/mongodb-org-8.0.repo >/dev/null <<'EOF'
  [mongodb-org-8.0]
  name=MongoDB Repository
  baseurl=https://repo.mongodb.org/yum/redhat/9/mongodb-org/8.0/x86_64/
  gpgcheck=1
  enabled=1
  gpgkey=https://pgp.mongodb.com/server-8.0.asc
  EOF
  ```

  ```bash
  sudo dnf install -y mongodb-mongosh
  ```

## Troubleshooting

- **Node does not appear on the Deployment page**: confirm the Agent is running (`sudo systemctl status mongodb-mms-automation-agent`) and that `mmsBaseUrl`, `mmsGroupId`, and `mmsApiKey` in `automation-agent.config` are correct. Check the node can reach `opsmgr-1` on 8080.
- **Deployment stalls before goal state**: confirm the new nodes can reach each other on 27017 (replica set members must connect directly), and that FQDNs resolve consistently across all hosts.
- **Hostname not listed when adding a member**: the Hostname menu lists only hosts running the Agent under this project. Verify the Agent installed and reported in on that node.
- **Permission errors starting mongod**: ensure the Agent's user owns the data and log directories with read/write access.
- **`mongosh: command not found` on a node**: Ops Manager installs the managed `mongod`
  binaries but does not put `mongosh` on your `PATH`. Use the bundled copy under
  `/var/lib/mongodb-mms-automation/mongosh-*/bin/mongosh`, or install the
  `mongodb-mongosh` package (see the Verification section above).
- **`getaddrinfo ENOTFOUND ...internal` when connecting with `?replicaSet=`**: this is
  expected, not a broken cluster. When you connect in replica set mode, `mongosh` uses
  the address you supply only to seed the connection, then reconnects using the member
  names from `rs.conf()`. Those names are whatever each node reports as its FQDN — in a
  cloud VPC that is often a private, internally-resolvable hostname (e.g.
  `ip-10-0-21-147.<region>.compute.internal`). A client outside the VPC (like your
  laptop) cannot resolve those names, so the connection fails even though the replica set
  is healthy. To connect from outside the VPC either:
  - run `mongosh` from inside the VPC (for example on one of the nodes), where the member
    names resolve, or
  - connect directly to a single node without replica set discovery, so it is not
    redirected to the internal names:

    ```bash
    mongosh "mongodb://<node-public-ip>:27017/?directConnection=true"
    ```

  If you need replica-set-mode connections to work from outside the VPC long term, the
  real fix is to configure the members with resolvable hostnames (public DNS names or
  entries every client can resolve) rather than private internal ones — a networking
  design choice beyond this test walkthrough.
