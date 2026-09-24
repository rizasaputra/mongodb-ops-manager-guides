# Enable Database-Level Audit Logging (Who Read/Wrote What)

This guide walks you through enabling MongoDB Enterprise database auditing on a replica set via Ops Manager, so you can see which users read and wrote which data at the database-operation level.

## Overview

You will turn on MongoDB's **database auditing** on the `new-rs0` replica set (Project A) through Ops Manager's deployment configuration, with a filter that records read and write operations and the user behind each. The audit log answers "who did what" at the level of actual database operations.

This is **auditing of MongoDB database operations**, not auditing of Ops Manager itself. It is a **MongoDB Enterprise** feature (Community cannot do it), which is why this project uses Enterprise binaries.

This guide targets `new-rs0` in Project A (from `10-provision-new.md`) — a test/lab deployment — so you can enable auditing and generate sample activity safely.

> **Performance and volume warning.** Capturing read/write operations requires enabling
> `auditAuthorizationSuccess`, which logs every successful authorization check. This can
> produce a very large audit volume and add overhead. Always pair it with a filter that
> narrows to the operations and namespace you care about. Do not enable broad
> read/write auditing on a busy production system without planning for the volume.

See the official docs for [Configure and Deploy Auditing](https://www.mongodb.com/docs/cloud-manager/tutorial/configure-auditing/) and [Configure Audit Filters](https://www.mongodb.com/docs/manual/tutorial/configure-audit-filters/) for reference.

## Key concepts

- **Audit destination / format / path** — where audit events go. This guide writes a
  **JSON file** on each node.
- **Audit filter** — a JSON predicate that limits which events are recorded. Without a
  filter, MongoDB records all auditable operations.
- **`atype`** — the audit event type. For read/write operations, the type is
  `authCheck`, and `param.command` names the operation (`find`, `insert`, `update`,
  `delete`, `findAndModify`).
- **`auditAuthorizationSuccess`** — a parameter that must be `true` to capture
  successful read/write operations. If left off (default), the audit only records
  authorization failures, not successful reads/writes.

## Prerequisites

- The `new-rs0` replica set running MongoDB 8.0 **Enterprise**, managed by Ops Manager (Automation) under Project A (from `10-provision-new.md`).
- Access control (authentication) enabled on the deployment, so audit events carry the user identity. This is what lets the log show *who* did each operation.
- Access to the Ops Manager console with rights to modify the deployment.
- A path on **each MongoDB node** (`new-1`, `new-2`, `new-3`) where the audit log can be
  written. The file is written by each `mongod`, so this is on the DB nodes — **not** on
  the Ops Manager host. `mongod` must be able to create/write the file, and it will
  **fail to start** if it cannot (`Unable to initialize audit path`). The safe choice is
  the **data directory the nodes already use**: `/data` (from `10-provision-new.md`) is
  owned by the `mongod` user and is already proven writable (the node writes
  `/data/mongodb.log` there), so this guide writes the audit log to `/data/mongodb-audit.json` —
  no directory creation or permission changes needed. Do **not** assume `/var/log/mongodb`
  exists: on these Automation-provisioned nodes it does not, and `mongod` cannot create it,
  so pointing the audit path there breaks startup.

## Steps

### 1. Open the deployment for editing

1. In the Ops Manager console, select Project A.
2. Click **Deployment** in the sidebar. Your `new-rs0` replica set is listed here (there is
   no separate "Clusters" tab in current Ops Manager).
3. On the `new-rs0` row, click the **ellipsis (…)** and select **Modify** (labelled **Edit
   Config** on some versions) to open its configuration.

### 2. Set the audit log destination, format, and path

In **Advanced Configuration Options**, add the audit output settings so they apply to
every `mongod` in the replica set:

- `auditLog.destination` — `file`
- `auditLog.format` — `JSON`
- `auditLog.path` — the audit log file path on each node: **`/data/mongodb-audit.json`**

These tell each `mongod` to write audit events as JSON to that file. `/data/mongodb-audit.json`
works with no preparation because `/data` already exists, is owned by the `mongod` user,
and is already writable by `mongod` (it holds `/data/mongodb.log`).

> **Do not point `auditLog.path` at a directory `mongod` cannot write.** `mongod` tries to
> create the audit path's directory on startup, and if it cannot it aborts with
> `Unable to initialize audit path` / `create_directory: Permission denied`, which fails
> the rolling deploy on that node. In particular, do **not** use `/var/log/mongodb/` on
> these Automation-provisioned nodes — it does not exist and `mongod` cannot create it. If
> you want a path other than `/data`, first create it on **every** node and give the
> `mongod` user ownership (`sudo mkdir -p /data/audit && sudo chown mongod:mongod /data/audit`),
> and on RHEL 9 confirm SELinux allows it (a path under the existing `/data` already does).

### 3. Add a filter that captures read and write operations

Add the audit filter as `auditLog.filter`. Use valid JSON with **double quotes around
every field name** (Ops Manager requires this; the MongoDB Agent can only read valid
JSON). To capture reads and writes against a specific collection, for example
`test.orders`:

```json
{ "atype": "authCheck", "param.command": { "$in": [ "find", "insert", "update", "delete", "findAndModify" ] }, "param.ns": "test.orders" }
```

- Drop `"param.ns"` to audit these operations across all namespaces (higher volume).
- `find` captures reads; `insert`, `update`, `delete`, and `findAndModify` capture writes.

### 4. Enable auditing of successful operations

`auditAuthorizationSuccess` is a **server parameter**, not one of the named `auditLog.*`
fields, so add it via `setParameter`:

**Advanced Configuration Options → Add Option → select `setParameter` → name
`auditAuthorizationSuccess`, value `true` → add it to the whole replica set.**

Without this, the log only shows authorization failures. With it, every successful
read/write authorization is logged — this is the setting that drives audit volume, so
keep the filter narrow.

### 5. Review and deploy

1. Click **Review & Deploy** (or **Confirm & Deploy**).
2. Ops Manager applies the change as a rolling restart of the replica set members, so the deployment stays available.

### 6. Generate some activity

Connect to the replica set and run a few operations against the audited namespace so the log has entries:

```bash
mongosh "mongodb://new-1.example.dev:27017/?replicaSet=new-rs0" -u <user> -p
```

```javascript
use test
db.orders.insertOne({ item: "audit-test", at: new Date() })
db.orders.find({ item: "audit-test" })
```

## Verification

- On any `new-rs0` node, the audit log file exists at the path you set (`/data/mongodb-audit.json`) and grows as you run operations.
- Inspect recent entries on a node:

  ```bash
  tail -n 20 /data/mongodb-audit.json
  ```

  Each line is one JSON audit event. Confirm your test insert and find appear.

### Read an audit entry

Two real entries from the `test.orders` filter — an insert (a write) followed by a find
(a read):

```json
{ "atype": "authCheck", "ts": { "$date": "2026-09-23T04:56:42.909+00:00" }, "local": { "ip": "10.0.21.147", "port": 27017 }, "remote": { "ip": "10.0.3.246", "port": 49946 }, "users": [], "roles": [], "param": { "command": "insert", "ns": "test.orders", "args": { "insert": "orders", "documents": [ { "item": "audit-test" } ], "$db": "test" } }, "result": 0 }
```

```json
{ "atype": "authCheck", "ts": { "$date": "2026-09-23T04:56:42.931+00:00" }, "local": { "ip": "10.0.21.147", "port": 27017 }, "remote": { "ip": "10.0.3.246", "port": 49946 }, "users": [], "roles": [], "param": { "command": "find", "ns": "test.orders", "args": { "find": "orders", "filter": { "item": "audit-test" }, "$db": "test" } }, "result": 0 }
```

Read each event by these fields — they answer who, when, and what:

- **What command** — `param.command`: `insert` in the first entry (a write), `find` in the
  second (a read). `param.args` shows the actual operation: the inserted document, or the
  find `filter` (`{ "item": "audit-test" }`).
- **On what data** — `param.ns`: `test.orders` (database `test`, collection `orders`).
- **When** — `ts` (`$date`): the UTC timestamp of the operation, down to milliseconds.
- **Who** — `users` and `roles`: the authenticated user and their roles. **In these
  samples both are empty (`[]`)** because the connection was not authenticated — so
  MongoDB cannot name the actor, only the source (`remote.ip` = `10.0.3.246`). To get a
  real "who", connect with credentials (see the note below); then `users` looks like
  `[{ "user": "<name>", "db": "admin" }]`.
- **Outcome** — `result`: `0` means the authorization check passed (the operation was
  allowed). A non-zero value is a failed/denied authorization.
- **Where from** — `remote.ip`/`port` is the client; `local.ip`/`port` is the node that
  served it (here `10.0.21.147:27017`).

> **Getting a real user in `users`.** Empty `users`/`roles` means the operation was
> unauthenticated. Connect with a user and auth database so the audit can attribute it, for
> example `mongosh "mongodb://new-1.example.dev:27017/?replicaSet=new-rs0" -u <user> -p --authenticationDatabase admin`,
> then rerun the operations. The new entries will carry that user in `users`.

## Troubleshooting

- **A node fails to start after deploy with `Unable to initialize audit path` / `create_directory: Permission denied`**: the `auditLog.path` points at a directory `mongod` cannot create or write. This is what happens if you set it to `/var/log/mongodb/mongodb-audit.json` on these nodes — that directory does not exist and `mongod` cannot create it. Fix it by setting `auditLog.path` to `/data/mongodb-audit.json` (the `mongod`-owned data directory) and redeploying; the stuck node then starts and the rolling restart completes. Check the cause on the node with `sudo tail -n 40 /data/mongodb.log`.
- **Audit log file is empty**: confirm `auditAuthorizationSuccess` is `true` (otherwise successful reads/writes are not recorded), that your filter matches the operations you ran, and that you ran operations against the audited namespace.
- **Agent logs a normalization error / filter ignored**: the filter must be valid JSON with double quotes around every field name. Re-check the `auditLog.filter` JSON.
- **No user shown on events**: enable access control (authentication) on the deployment; without it, MongoDB cannot attribute operations to a user.
- **Log grows too fast**: narrow the filter (add `param.ns`, or limit `param.command`), or reconsider whether you need `auditAuthorizationSuccess` for your use case.
- **Auditing options not available**: confirm the deployment runs MongoDB **Enterprise** — auditing is not available in Community.
