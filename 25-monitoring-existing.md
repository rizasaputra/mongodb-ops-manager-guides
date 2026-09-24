# Monitor an Existing Production System and Use Performance Advisor

This guide walks you through attaching Ops Manager to your existing, already-running production replica set to monitor its metrics and use the Performance Advisor on real production traffic.

## Overview

You will add the existing production replica set to Ops Manager as a new project (Project B — Existing), watch its live metrics, and use the Performance Advisor to review slow queries and suggested indexes based on the real workload already running against it.

The Performance Advisor evaluates the slow queries in your deployment's logs and suggests indexes, grouped by query shape, that would improve them. It does not add load of its own.

> **Production warning.** Project B is your real production system. Two things follow
> from that:
>
> - **Do not generate dummy data or inject synthetic slow queries.** The whole point
>   here is to analyze the workload that is already running. Synthetic load belongs on
>   a separate test deployment, never on production.
> - **Attaching Ops Manager changes production hosts.** Installing the MongoDB Agent,
>   and especially enabling Automation, touches the running nodes. Automation import
>   can restart the processes in the project, so do this in an approved change window.

This guide assumes you completed `00-install-ops-manager.md` and have a running Ops Manager on `opsmgr-1`. See the official docs for [Add Existing MongoDB Processes to Ops Manager](https://www.mongodb.com/docs/ops-manager/current/tutorial/add-existing-mongodb-processes/) and [Monitor and Improve Slow Queries](https://www.mongodb.com/docs/ops-manager/current/tutorial/performance-advisor/) for reference.

## Monitoring-only vs. Automation

You can add an existing deployment for **Monitoring only** or for **Monitoring and Automation**. This choice matters, and it is a trade-off:

- **Monitoring only** is the least invasive: the Agent reads metrics and logs but does not manage or restart your processes. It does not, however, enable the Performance Advisor.
- **Automation** lets Ops Manager manage the deployment (config changes, restarts) — but importing to Automation can restart every process in the project to apply the project's security settings, and it may change users/roles. This is a bigger, production-impacting change.

Here, **the Performance Advisor requires the cluster to be managed by MongoDB Agent Automation**, so this guide adds the deployment directly to Monitoring and Automation. Because that is production-impacting, do it deliberately in an approved change window.

## Prerequisites

- A running Ops Manager on `opsmgr-1` (from `00-install-ops-manager.md`).
- The existing production replica set, healthy, running MongoDB 8.0 Enterprise.
- An approved change window and sign-off to attach Ops Manager to production.
- Network access:
  - Each production node can reach `opsmgr-1` on 8080/8443.
  - `opsmgr-1` can reach each production node on 27017.
- Credentials for the production deployment if it uses authentication (it should). You will need to supply the MongoDB Agent user credentials during import.
- Time synchronized (`chronyd`) on the production nodes (see `00-install-ops-manager.md`).

## Steps

### 1. Create a separate project for the existing system

Create a dedicated project for the production deployment so its security settings and
users stay isolated from `Project A - New`. You must be an Organization Owner or
Organization Project Creator to create a project; you become its Project Owner.

1. In the navigation bar, select the organization that will contain the project from
   the **Organizations** menu.
2. Click the leaf icon in the upper-left corner, then click **New Project**. (You can
   also expand the **Projects** menu in the navigation bar and click **+ New Project**.)
3. **Name** the project, for example `Project B - Existing`.
4. For **Default Project Server Type**, choose **Production Server**, since this project
   manages your real production deployment.
5. Add project members if prompted (enter the username/email of existing Ops Manager
   users, or an email address to invite a new one). You are added as Project Owner
   automatically.
6. Finish creating the project. Ops Manager assigns a set of default alert
   configurations to it automatically.

### 2. Create the automation user on the production cluster

Ops Manager manages the deployment as the `mms-automation` MongoDB user, so it must
exist before you import. Connect to the primary of your production replica set and run:

```javascript
db.getSiblingDB("admin").createUser({
  user: "mms-automation",
  pwd: "<agent-user-password>",
  roles: [
    "clusterAdmin",
    "dbAdminAnyDatabase",
    "readWriteAnyDatabase",
    "userAdminAnyDatabase",
    "restore",
    "backup",
    "directShardOperations"  // only used by sharded clusters; harmless on a replica set
  ]
})
```

- `<agent-user-password>` — a strong password you choose; supply this same value to the wizard in step 5.
- `directShardOperations` is only used by sharded clusters; it is harmless on a replica set and kept so the same user works everywhere.

Note: on import, Ops Manager applies its own keyfile to the deployment, which triggers a rolling restart (one member at a time). This is expected — do the import in your change window.

### 3. Launch the Add Existing wizard

1. Select your organization and Project B.
2. Click **Deployment** in the sidebar.
3. Start the **Add Existing** wizard to add existing MongoDB processes.

The wizard prompts you to install the MongoDB Agent (the production nodes do not have it yet) and to identify the replica set to add. The wizard shows the exact install commands for your project — the next step covers what those commands do.

### 4. Install the MongoDB Agent on each production node

The production nodes are already running `mongod` but have no MongoDB Agent yet. Install it on every production node, as `root` or with `sudo`.

First, generate an Agent API Key for Project B. The Agent needs one key per project to talk to Ops Manager:

1. With Project B selected, click **Deployment**, then **Agents**, then **Agent API Keys**.
2. Click **Generate**, enter a description, and click **Generate** again.
3. Copy the key immediately and store it securely. Ops Manager shows the full key only once — treat it like a password.

You will also need the **Project ID** for Project B, shown in the Ops Manager console.

> Because these hosts already run MongoDB, follow these rules from the official docs:
>
> - **Use the same package manager you used for MongoDB.** Install the Agent with the
>   same package manager (here, `dnf`/`rpm`) so the Agent shares MongoDB's owner and
>   file permissions.
> - **The Agent user needs read/write on the existing `mongod` data and log
>   directories.**

The Agents page in your project gives the exact URL and version. The command looks like:

```bash
curl -OL http://<opsmgr-host>:8080/download/agent/automation/mongodb-mms-automation-agent-manager-<version>.x86_64.rhel9.rpm
```

```bash
sudo rpm -U mongodb-mms-automation-agent-manager-<version>.x86_64.rhel9.rpm
```

Set your project credentials in `/etc/mongodb-mms/automation-agent.config`:

```text
mmsBaseUrl=http://<opsmgr-host>:8080
mmsGroupId=<project-id>
mmsApiKey=<agent-api-key>
```

- `<opsmgr-host>` — hostname or IP of `opsmgr-1`.
- `<version>` — the Agent version shown on the Agents page.
- `<project-id>` — the Project B ID.
- `<agent-api-key>` — the Agent API Key you generated for Project B above.

Disable the existing `mongod` service so systemd does not race the Agent to start `mongod` on reboot once Automation manages it:

```bash
sudo systemctl is-enabled mongod.service
```

```bash
sudo systemctl disable mongod.service
```

Then start the Agent:

```bash
sudo systemctl enable --now mongodb-mms-automation-agent
```

Repeat on every production node. Within a minute or two, each node reports in to Ops Manager under Project B.

### 5. Add the deployment to Monitoring and Automation

Back in the Add Existing wizard, when it asks whether to add the deployment to **Monitoring** or **Monitoring and Automation**, choose **Monitoring and Automation**. Automation is required for the Performance Advisor, which is the goal of this scenario.

1. Identify the deployment. Because the Agent is already running on the nodes (step 4),
   the wizard detects the running `mongod` processes. Select the replica set to add, or
   enter a seed member as `hostname:port` (a node's FQDN and port, e.g.
   `prod-1.example.dev:27017`) so Ops Manager can discover the rest of the set.
2. When prompted, supply the `mms-automation` user and password you created in step 2 so
   Ops Manager can authenticate to the deployment.

Because you added the `mms-automation` user to the source in step 2, the import can authenticate. Ops Manager then applies its own automation keyfile, which triggers a rolling restart of the processes in the project (as noted in step 2). Do the import in your approved change window.

### 6. Confirm the deployment appears and watch metrics

1. On the **Deployment** page for Project B, confirm the production replica set appears with its members healthy.
2. Open the deployment and review live metrics: opcounters, operation execution time, connections, replication lag, and cache/memory. These reflect the real production workload.

Let it collect for a while so the metrics and slow-query logs represent normal production activity.

### 7. Open the Performance Advisor

With the deployment managed by Automation:

1. Open Project B, then the production replica set.
2. Open the **Performance Advisor**.
3. Review the suggested indexes. Each is ranked by **Impact** (High or Medium) based on wasted bytes read, and comes with sample query shapes and metrics such as Execution Count, Average Execution Time, and Average Query Targeting.

Because this runs against real production traffic, the query shapes and suggestions reflect actual slow operations your workload is producing — no synthetic data needed.

### 8. Review, do not blindly apply, index suggestions

The Performance Advisor can create a suggested index directly, but on production treat every suggestion as a proposal:

- Weigh the read/write trade-off. Indexes speed reads but cost writes; a busy write-heavy collection may not want another index.
- Check whether an existing index or a query change would solve it instead.
- Applying an index on production is itself a change — schedule it, and prefer a rolling index build to reduce impact.

## Verification

- Project B shows the production replica set with all members healthy and live metrics updating.
- After enabling Automation and letting it run, the **Performance Advisor** page lists query shapes and suggested indexes drawn from the real workload (it evaluates up to the most recent 20,000 slow queries / 200,000 log lines).
- You did not create any test data or run synthetic queries against production.

## Troubleshooting

- **Performance Advisor is empty or unavailable**: confirm the cluster is managed by **Automation** (monitoring-only is not enough), that MongoDB is 3.2+ (yours is 8.0), and that your Ops Manager user has a role permitted to view example query field values. An empty list can also simply mean no slow queries have been logged yet.
- **Deployment does not appear after import**: confirm the Agent is installed and reporting, the nodes can reach `opsmgr-1` on 8080/8443, and you supplied correct monitoring/automation credentials.
- **Import to Automation stalls or errors on auth**: confirm you completed step 2 — the `mms-automation` user exists on the source `admin` database with the listed roles, and you supplied the matching password to the wizard.
- **Restart during Automation import**: this is expected. Ops Manager applies its own automation keyfile on import, which restarts the processes in the project (see step 2). Only do the import in an approved change window.
