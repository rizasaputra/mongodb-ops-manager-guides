# Change a Runtime Parameter with No Downtime

This guide walks you through using Ops Manager automation to change a running MongoDB parameter — the slow-query threshold — and confirming the replica set stays available the whole time.

## Overview

You will change the `slowOpThresholdMs` parameter on the replica set you deployed in `10-provision-new.md`, using the Ops Manager console. Ops Manager applies the change as a **rolling restart**: it updates one member at a time and always keeps a primary available, so the replica set stays online. During the restart the primary steps down and the set elects a new one — you can watch the **PRIMARY** move on the Deployment page. You will run a continuous write loop during the change; with retryable writes on (the default), writes keep succeeding through the failover. A brief latency bump on one write is possible, but with a fast election it may not be visible at all.

`slowOpThresholdMs` sets the threshold, in milliseconds, above which an operation is logged as slow (and, with profiling on, recorded in the profiler). The default is 100 ms.

This guide assumes you completed `10-provision-new.md` and have a healthy three-member replica set (`new-rs0`) under Project A. See the official docs for [Edit a Deployment's Configuration](https://www.mongodb.com/docs/ops-manager/current/tutorial/edit-deployment/) and [Restart Processes](https://www.mongodb.com/docs/ops-manager/current/tutorial/restart-processes/) for reference.

## Prerequisites

- A healthy three-member replica set `new-rs0` under Project A, all members at goal state (from `10-provision-new.md`).
- Access to the Ops Manager console with rights to modify the deployment.
- `mongosh` available on a host that can reach the replica set, to run the availability check.
- The replica set connection string, e.g. `mongodb://new-1.example.dev:27017,new-2.example.dev:27017,new-3.example.dev:27017/?replicaSet=new-rs0`.

## Steps

### 1. Start a continuous write loop (availability check)

Before making the change, start a loop that writes once per second and prints whether each write succeeded. Leave this running in a separate terminal for the whole change so you can watch availability.

Connect with `mongosh` using the full replica set connection string so the driver can fail over automatically:

```bash
mongosh "mongodb://new-1.example.dev:27017,new-2.example.dev:27017,new-3.example.dev:27017/?replicaSet=new-rs0"
```

In the `mongosh` shell, run this loop. It records how long each write takes and prints
the latency, so a failover shows up as a longer-than-usual write rather than an error:

```javascript
while (true) {
  const start = Date.now();
  try {
    db.getSiblingDB("downtime_test").pings.insertOne({ t: new Date() });
    print(new Date().toISOString() + "  write OK  (" + (Date.now() - start) + " ms)");
  } catch (e) {
    print(new Date().toISOString() + "  WRITE FAILED: " + e.message);
  }
  sleep(1000);
}
```

You should see a steady stream of `write OK` with low latency (typically single- to
low-double-digit milliseconds). Keep this running.

**About retryable writes:** MongoDB enables retryable writes by default
(`retryWrites=true`) with modern drivers, and `mongosh` uses one. When the primary
steps down during the rolling restart, the driver automatically retries the in-flight
write once against the newly elected primary. As a result you will most likely see the
write **succeed** with a higher latency (the time spent waiting for the election)
rather than a `WRITE FAILED` line. That elevated latency is the visible signature of
the failover.

### 2. Open the deployment for editing

In the Ops Manager console:

1. Select your organization and Project A.
2. Click **Deployment** in the sidebar. The deployment list shows your clusters and
   replica sets, including `new-rs0`. 
3. On the `new-rs0` row, click **Modify** button (some versions label this **Edit Config**). This opens the replica set's configuration.

### 3. Set the slow-query threshold

1. Scroll to **Advanced Configuration Options**.
2. Click **Add Option**.
3. Select the `slowOpThresholdMs` option.
4. Enter the new value, for example `200` (log operations slower than 200 ms).
5. Add the option so it applies to every mongod process in the replica set. Click **Save**.

### 4. Review and deploy

1. Click **Review & Deploy** then **Confirm & Deploy**.

Ops Manager now applies the change with a rolling restart: it updates the members one at a time and always keeps a primary available. During the restart a member may briefly step down, triggering a fast election, but the replica set as a whole stays writable.

### 5. Watch the write loop during deployment

As Ops Manager restarts each member, the rolling restart forces the current primary to
step down and the replica set to elect a new one. The clearest proof of this is on the
**Deployment** page: watch the **PRIMARY** badge move from one member to another while the
set stays writable. That failover is the whole point — it happens with no downtime.

Meanwhile, switch to the terminal running the write loop from step 1. You *may* catch one
write reporting a higher latency (a few hundred milliseconds to a few seconds) while the
driver retries against the newly elected primary. But do not be surprised if you see no
blip at all: elections in a small healthy replica set are fast, and with a 1-second loop
the failover often completes between two writes, so no single write lands on it. Either
way the loop keeps printing `write OK` — because retryable writes are on by default, a
failover becomes a slightly slower `write OK` rather than a `WRITE FAILED`.

This is the "no downtime" demonstration: the primary changed over (visible on the
Deployment page) and every write still succeeded through the configuration change. A
visible latency bump is possible but not guaranteed.

## Verification

- On the **Deployment** page, `new-rs0` returns to **goal state reached** with one **PRIMARY** and two **SECONDARY** members. During the rolling restart you should have seen the **PRIMARY** move to a different member — that is the failover, and the main proof of no downtime.
- The write loop from step 1 kept printing `write OK` throughout. You may or may not see a single elevated-latency write during the failover; a fast election with a 1-second loop often produces no visible blip. Stop it with `Ctrl+C`.
- Confirm the parameter actually changed. `slowOpThresholdMs` is not readable via
  `getParameter` (that returns `InvalidOptions: no option found to get`); it is the slow
  operation threshold, so read it back through the profiling status instead. Connect and
  check `slowms`:

  ```bash
  mongosh "mongodb://new-1.example.dev:27017/?replicaSet=new-rs0"
  ```

  ```javascript
  db.getProfilingStatus()
  ```

  The output should show `slowms: 200` (the value you set). Ops Manager applies the option
  to every process, so any database's profiling status reports the same `slowms`.

- Optionally, clean up the test data:

  ```javascript
  db.getSiblingDB("downtime_test").dropDatabase()
  ```

## Troubleshooting

- **Write loop shows sustained `WRITE FAILED` lines (not just one slow write)**: retryable writes should turn a single failover into a slow `write OK`, so repeated failures mean something is wrong. Check the **Deployment** page for a member stuck off goal state. A healthy rolling restart keeps a majority available; sustained failures suggest more than one member is down at once. Confirm all three members were healthy before you started.
- **Change does not deploy**: make sure you clicked **Confirm & Deploy** and that all Agents are reporting in. A change stays pending until every relevant Agent reaches goal state.
- **`getParameter` shows the old value**: confirm the deploy finished (goal state reached) and that you added the option to the whole replica set, not a single process you did not check.
- **Cannot connect with `mongosh`**: verify the connection string hostnames and that the client host can reach the members on 27017.
