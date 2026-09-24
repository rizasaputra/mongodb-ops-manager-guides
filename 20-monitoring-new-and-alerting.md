# Monitor the New Replica Set, Performance Advisor, and Alerting

This guide walks you through watching metrics for the `new-rs0` replica set, generating slow queries so the Performance Advisor has something to recommend, and setting up an Ops Manager alert that notifies a Telegram bot.

## Overview

You will use Project A's `new-rs0` replica set (from `10-provision-new.md`) — a test/lab deployment — to:

- Review live monitoring metrics in Ops Manager.
- Generate fake slow queries and see the Performance Advisor surface index suggestions.
- Create an alert with a deliberately low threshold.
- Deliver the alert to a Telegram bot via an Ops Manager webhook.
- Trigger the alert by generating a little activity, then watch it open and close.

Because this is a lab deployment, it is safe to generate synthetic slow queries and activity here — something you would never do against production (that is why the production guide, `25-monitoring-existing.md`, forbids it and relies on real traffic instead).

This guide assumes Ops Manager is running and `new-rs0` is deployed and managed by Automation under Project A. See the official docs for [Monitor and Improve Slow Queries](https://www.mongodb.com/docs/ops-manager/current/tutorial/performance-advisor/) and [Configure Alert Settings](https://www.mongodb.com/docs/ops-manager/current/tutorial/manage-alert-configurations/) for reference.

## How alerting fits together

Ops Manager alerting has two parts:

- **Alert configuration** — the condition that opens an alert (metric + threshold), plus how long the condition must last before it notifies.
- **Notification method** — where the alert goes: email, Slack, PagerDuty, webhook, and others.

We use the **webhook** method to reach Telegram.

## Reshaping the payload for Telegram

By default Ops Manager's webhook POSTs its own alert JSON (the same format as the API
Alerts resource) with an event header. Telegram's Bot API expects something different — a
POST to `https://api.telegram.org/bot<token>/sendMessage` with a body of
`{ "chat_id": ..., "text": ... }`. So the payload has to be reshaped.

Ops Manager reshapes it with **FreeMarker templates**: in the per-alert **Add Webhook**
dialog you set a **Body Template** and **Header Template** that emit exactly the JSON
Telegram expects. So you point the webhook straight at Telegram's `sendMessage` endpoint —
no relay or extra service required.

## Prerequisites

- Ops Manager running, with `new-rs0` deployed and managed under Project A (from `10-provision-new.md`).
- `mongosh` on a host that can reach `new-rs0`, to generate activity.
- A Telegram bot: create one with **BotFather** to get a **bot token**, and get your **chat_id** (send the bot a message, then read `https://api.telegram.org/bot<token>/getUpdates` and take `result[].message.chat.id`).
- **Outbound HTTPS from `opsmgr-1` to `api.telegram.org`** — Ops Manager posts directly to
  Telegram.
- Webhook Body/Header Template fields available in the per-alert **Add Webhook** dialog.

## Steps

### 1. Enable Monitoring on each node

Automation only deploys and runs the `mongod` processes. Metrics, Ping, and the green
status dots come from the MongoDB Agent's **Monitoring** function, which is separate and
may not be on yet. If `new-rs0` shows grey processes with empty Ping, "No Data Available",
or a "No Monitoring detected" banner, the deployment is fine — you just need to turn
Monitoring on. Do this first, because every other step here (metrics, Performance
Advisor, alerts) depends on monitoring data flowing.

You activate Monitoring per host from the Servers view. You do not reinstall anything; you
are telling the Agent that is already running to start collecting and reporting metrics.

1. In the Ops Manager console, select Project A, then click **Deployment** and open the
   **Servers** tab. Your three nodes are listed here.
2. On each host, click its actions menu (the **⋯** / gear icon on that host's row) and
   click **Activate Monitoring**. Do this for all three nodes now — Ops Manager stages the
   changes rather than applying them immediately, so you can queue up all three before
   deploying. Activating Monitoring on more than one host is good practice: Ops Manager
   makes one Agent the primary Monitor and keeps the others on standby for failover.
3. Once all three are staged, click **Review & Deploy** in the banner, then **Confirm &
   Deploy** once to apply them all in a single pass.

After a minute or two, the **Servers** tab shows one host as **Monitoring - active** and
the others as **Monitoring - standby**, and on the **Processes** view the status dots for
`new-rs0` turn **green** with Ping and metrics populated. If they are still grey, see
Troubleshooting.

### 2. Watch the monitoring metrics

1. In the Ops Manager console, select Project A and open `new-rs0` on the **Deployment** page.
2. Review live metrics: opcounters, operation execution time, connections, replication lag, cache/memory. Confirm data is flowing — this is the monitoring the alert will act on.

### 3. Generate fake slow queries

The Performance Advisor recommends indexes based on **slow queries** in your deployment —
operations that run longer than the slow query threshold (`slowOpThresholdMs`, default
**100 ms**). `mongod` writes every operation over that threshold to its slow query log
automatically (you do not need to enable the profiler for this). On a brand-new replica
set nothing is slow yet, so you generate some.

Two things make a query reliably cross 100 ms on a lab node:

- **Enough data, with padding.** A full collection scan is slow when it has to read a lot
  of bytes, so seed a couple hundred thousand documents and pad each one so the scan moves
  real data.
- **An unindexed sort over many matches.** Matching many documents and sorting them on a
  field with **no index** forces a large in-memory sort on top of the collection scan.

#### 3a. Seed a padded collection

Connect to the replica set (this routes writes to the primary) and seed `perftest.users`.
Each document carries a ~2 KB `pad` string so scans are expensive:

```bash
mongosh "mongodb://new-1.example.dev:27017/?replicaSet=new-rs0"
```

```javascript
use perftest
db.users.drop();   // start clean if you seeded before
const pad = "x".repeat(2000);   // ~2 KB per document
let bulk = [];
for (let i = 0; i < 200000; i++) {
  bulk.push({ i, email: "user" + i + "@example.dev", score: Math.random(), pad });
  if (bulk.length === 5000) { db.users.insertMany(bulk); bulk = []; }
}
db.users.countDocuments();   // expect 200000
```

#### 3b. Confirm the query is actually slow

Before looping, verify the query crosses 100 ms with `explain`. This query matches about
half the documents (`score > 0.5`) and sorts them by `email`, which has no index — a big
unindexed sort:

```javascript
db.users.find({ score: { $gt: 0.5 } }).sort({ email: 1 }).explain("executionStats").executionStats.executionTimeMillis
```

You want a value comfortably above 100 (a few hundred milliseconds is typical for this
shape). If it is borderline on your hardware, make the scan heavier: increase the padding
(e.g. `"x".repeat(6000)`, re-seed with a smaller batch like 2000 to stay under the 16 MB
insert limit) or seed more documents.

#### 3c. Run the slow queries in a loop

Run the verified slow query a handful of times so the Advisor sees a few executions per
shape. Both shapes below are **improvable** — they filter/sort on unindexed fields, so the
Advisor can suggest an index for them. Keep the count small: each iteration is a few
hundred milliseconds, so even 20 rounds takes a while:

```javascript
// Unindexed fields, so each of these does a full collection scan plus an in-memory sort.
for (let n = 0; n < 20; n++) {
  db.users.find({ score: { $gt: 0.5 } }).sort({ email: 1 }).toArray();
  db.users.find({ email: "user199999@example.dev" }).toArray();
}
```

That is plenty for the Advisor to flag these query shapes. They scan far more documents
than they return, which is exactly what it looks for. Optionally confirm they are landing
in the slow query log on the primary:

```bash
sudo grep -i 'Slow query' /data/mongodb.log | tail -n 5
```

### 4. Review the Performance Advisor

The Performance Advisor requires the cluster to be managed by Automation — `new-rs0` is,
from `10-provision-new.md`.

1. In Project A, open `new-rs0`, then open the **Performance Advisor**.
2. Give it a few minutes. The Agent collects slow-query data on an interval and the
   Advisor analyzes it, so suggestions do not appear instantly — expect roughly 5-10
   minutes after the queries run, sometimes longer. (If nothing ever appears, the usual
   cause is that the queries were not actually over 100 ms — recheck with the `explain`
   in step 3b.)
3. Once analyzed, the **Create Indexes** view lists **suggested indexes**, one card per
   collection/query shape (for example, an index on `perftest.users` covering
   `email`/`score`). The list is **sorted by Impact** — you will see a "SORTED BY: IMPACT"
   label at the top right — meaning the highest-impact suggestion is listed first. Impact
   only orders the list here; it is not shown as a per-card "High/Medium" label.
4. Each suggestion card shows the metrics that tell you whether the index is worth it:
   **Execution Count** (per hour), **Average Execution Time**, **Average Query Targeting**
   (docs read per doc returned — higher means more wasteful), **In Memory Sort**, Avg Docs
   Scanned/Returned, and Avg Object Size. A high Average Query Targeting is the clearest
   sign the index will help — for the unindexed queries you ran, expect it well above 1.
   Expand **Sample Queries Improved By This Index** to see the actual query shapes.
5. Optionally, click **Create Index** on a suggestion to build it directly from the
   Advisor, then re-run the queries from step 3 and confirm that query shape drops off the
   suggestions and its execution time falls.

Since this is a lab, creating suggested indexes here is safe. (On production you would
treat each suggestion as a proposal and weigh the read/write trade-off first.)

> **Don't follow the Advisor blindly.** The Performance Advisor's suggestions are
> proposals, not ground truth — the exact fields and their order are not always optimal
> for the query you care about. Evaluate each suggestion against indexing best practices,
> especially the **ESR rule** (index fields in the order **Equality, Sort, Range**). For
> example, for `find({ score: { $gt: 0.5 } }).sort({ email: 1 })` the ESR-optimal index is
> `{ email: 1, score: 1 }` (sort field before the range field, so MongoDB walks the index
> in order and avoids an in-memory sort) — which may differ from what the Advisor lists.
> Confirm an index is actually used with `explain()` (look for `IXSCAN` and no `SORT`
> stage), not just by watching the execution time. See the MongoDB manual on the
> [ESR rule](https://www.mongodb.com/docs/manual/tutorial/equality-sort-range-guideline/)
> and [indexing strategies](https://www.mongodb.com/docs/manual/applications/indexes/).

### 5. Create the alert with a low threshold

Open the alert form: in Project A, click the **Project Alerts** icon in the navigation bar
(or **Alerts** in the sidebar), then click **Add** and select **New Alert**. The form has
three sections — **Alert if**, **For**, and **Send to** — described below.

#### 5a. Alert if — pick the target, then the metric

This is the part that tripped you up. The **first dropdown is not the metric — it is the
alert _target_ (the kind of component being watched)**: Host, Replica Set, Sharded
Cluster, Agent, Backup, User, Project, and so on. The list of available conditions/metrics
in the **second** dropdown changes depending on which target you pick. There is no
"Connections" until you select the right target first.

Per-`mongod` performance metrics like connections and query targeting live under the
**Host** target:

1. In the first dropdown, select **Host** (labelled something like "Host" / "Host has").
2. In the metric dropdown that appears, choose one that is easy to cross in a lab:
   - **Connections** — fires when the current connection count on a host is above a
     number. Set the threshold to a small value like **above `5`**. The baseline is
     already well above this (replica set members and the monitoring agent hold dozens of
     connections on their own), so the alert opens right away and stays open. That is fine
     for this lab — we just want to see it fire and get delivered; we do not care that it
     never clears.
   - or **Query Targeting: Scanned Objects / Returned** — the ratio of documents scanned
     to documents returned. This one only spikes on unindexed scans, so use it if you want
     an alert that opens and later clears. Set the threshold to a low ratio (for example,
     above `100`).
3. When you pick **Host**, Ops Manager also asks which host **type** the alert applies to
   (Of any type, Primary, Secondary, Arbiter, Standalone). For this lab choose **Primary**
   so exactly one host matches and you get a single alert. "Of any type" would match all
   three members and open three separate alerts.
4. Set the comparison to **is above** and enter the threshold.

If you cannot find "Connections", double-check that the target is **Host** and not
Replica Set or Project — the metric list is different for each target.

#### 5b. For — optionally narrow the scope

The **For** section (if shown) lets you filter which targets the alert applies to, using a
logical OR between conditions, and the match field accepts regular expressions. For this
lab just leave the default `Any Host`.

#### 5c. Send to — deliver straight to Telegram via webhook

1. In the **send if condition lasts at least** field, enter a small value (for example
   `1` minute) so the alert fires quickly instead of waiting the default window.
2. Click **Add** and choose **Webhook** as the notification method. The dialog expands to
   show the webhook fields.
3. Fill in the fields to post directly to Telegram:
   - **Webhook URL** — the Telegram `sendMessage` endpoint with your bot token in the path:

     ```text
     https://api.telegram.org/bot<bot-token>/sendMessage
     ```

   - **Webhook Body Template** — a FreeMarker template that emits Telegram's JSON. The `!`
     defaults guard against null fields (which otherwise cause render errors), and
     `?js_string` safely escapes the text for JSON:

     ```json
     {
       "chat_id": "<chat-id>",
       "text": "[${event!'alert'}] ${humanReadable?js_string}"
     }
     ```

   - **Webhook Header Template** — set the content type so Telegram parses the body:

     ```json
     { "Content-Type": "application/json" }
     ```

   - **Webhook Secret** — leave blank. (It only makes Ops Manager add an `X-MMS-Signature`
     header for you to verify; Telegram ignores it.)

   Replace `<bot-token>` and `<chat-id>` with your values. Both templates must produce
   valid JSON or Ops Manager rejects them when you save.
4. If a **Post Test Alert** / test link is offered, click it — Ops Manager renders the
   template with sample data and posts it to Telegram, so a message should arrive in your
   chat. (Some Ops Manager versions do not offer a manual test; if yours does not, you
   verify by triggering the real condition in step 7.)
5. Click **Save** (or **Add Alert**) to create the alert. It now appears on the **Alert
   Settings** tab, enabled.

### 6. Trigger the alert with activity

Generate light activity on `new-rs0` to cross the threshold. If you used a **Connections**
alert with a low threshold, simply opening a few `mongosh` sessions may already trip it. If
you used **Query Targeting**, run some unindexed queries in a loop to push the ratio up:

```bash
mongosh "mongodb://new-1.example.dev:27017/?replicaSet=new-rs0"
```

```javascript
use alerttest
for (let i = 0; i < 1000; i++) db.t.insertOne({ i, v: Math.random() });
// unindexed scan, repeated, to push query-targeting up
for (let i = 0; i < 200; i++) db.t.find({ v: { $gt: 0.999 } }).toArray();
```

Keep activity going past the "lasts at least" window you set.

## Verification

- **Monitoring**: on the **Servers** tab one host reads **Monitoring - active** and the
  others **Monitoring - standby**, and `new-rs0` shows green processes with live metrics
  updating on the Deployment/Processes page.
- **Performance Advisor**: after the slow queries in step 3, the Performance Advisor lists suggested indexes (e.g. on `email` and `score`) with per-suggestion metrics (Execution Count, Average Execution Time, Average Query Targeting). If you created a suggested index, that query shape drops off after re-running the queries.
- **Delivery**: the webhook **Post Test Alert** in step 6c produced a message in your
  Telegram chat (if your version offers the test button).
- **Real alert**: after generating activity, the alert appears on the **Alerts > Open** tab in Ops Manager, and your Telegram chat receives a message (Ops Manager sends the `alert.open` state).
- **Resolution**: stop the activity; once the metric falls back under the threshold, Ops Manager closes the alert (it leaves the **Open** tab) and sends the `alert.close` state to Telegram.
- Clean up the test data if you like:

  ```javascript
  db.getSiblingDB("alerttest").dropDatabase()
  ```

## Troubleshooting

- **Processes stay grey / "No Monitoring detected" / "No Data Available" after activating**:
  give it a couple of minutes for the first metrics to arrive. If it persists, confirm on
  the **Servers** tab that at least one host shows **Monitoring - active** (if all say
  standby or none is active, re-run **Activate Monitoring** and **Review & Deploy > Confirm
  & Deploy**). Also confirm the nodes can reach Ops Manager on port 8080, since the Agent
  pushes monitoring data back to Ops Manager. The replica set itself can be perfectly
  healthy (`rs.status()` shows a PRIMARY and SECONDARYs) while monitoring is still off —
  grey dots are a monitoring signal, not a `mongod` health signal.
- **Performance Advisor shows no recommendations**: first give it time — data collection
  and analysis take several minutes. If it stays empty, your queries were most likely
  under the 100 ms slow query threshold, so nothing was logged as slow. Confirm with
  `db.users.find({ score: { $gt: 0.5 } }).sort({ email: 1 }).explain("executionStats").executionStats.executionTimeMillis`
  (want > 100), and make the scan heavier if needed (more padding or more documents). Also
  make sure you ran the queries against the **primary** (slow ops are logged on the node
  that executed them) and that the Advisor is scoped to `new-rs0`. Note the slow query log
  is always on, but the profiler is not — you can check the active threshold with
  `db.getProfilingStatus()` (it reports `slowms`).
- **Ops Manager rejects the template on save**: both the Body and Header templates must
  produce **valid JSON**. A common cause is an unguarded null field — use the `!` default
  form (`${event!'alert'}`, `${humanReadable!''}`) so missing fields do not break
  rendering, and `?js_string` on free text so quotes/newlines are escaped.
- **No Telegram message at all**: sanity-check the endpoint by hand — `curl -X POST "https://api.telegram.org/bot<bot-token>/sendMessage" -H "Content-Type: application/json" -d '{"chat_id":"<chat-id>","text":"test"}'` should post to your chat. Then confirm `opsmgr-1` has outbound HTTPS to `api.telegram.org`, and that you started a chat with the bot (a bot cannot message a user who never messaged it).
- **Telegram rejects the request**: verify the bot token and the `chat_id` from `getUpdates`, and confirm the rendered Body Template is valid JSON with `chat_id` and `text`. The Header Template should set `Content-Type: application/json`.
- **Alert never opens**: the condition may not have crossed the threshold, or not for long enough. Lower the threshold, generate more activity, and confirm the alert is enabled on the **Alert Settings** tab.
- **Alert opens but nothing is sent**: confirm the alert's notification method is **Webhook** and that its Webhook URL and Body/Header templates are filled in.
