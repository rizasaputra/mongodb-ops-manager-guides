# Testing Environment & Scenarios

Context for the Ops Manager testing guides in this project. All guides describe a
test/lab environment used to exercise Ops Manager features end to end, so a reader
can follow them without going back to the MongoDB docs.

## Baseline versions and platform

- **Ops Manager**: 8.0 (use 8.0.8 or later; third-party backup integration with
  Veritas/Cohesity requires 8.0.8+)
- **MongoDB**: 8.0 (major release; do not use minor releases for backing databases)
- **MongoDB edition**: Enterprise binaries for all scenarios (required for database
  auditing and other enterprise features).
- **Operating system**: RHEL 9.4.

## Ops Manager deployment model

- Single Ops Manager host running in **test mode** (single host, no HA, backup enabled
  as needed for the scenarios). This is an evaluation setup, not production.
- One Ops Manager instance hosts **two projects**:
  - **Project A — New**: MongoDB provisioned from scratch onto empty nodes by
    Ops Manager automation. See `10-provision-new.md`.
  - **Project B — Existing (PRODUCTION)**: the real, already-running production
    MongoDB replica set (v8, Enterprise) that Ops Manager is attached to for
    monitoring, performance advisor, and backup. This is live production, not a lab
    stand-in. Treat every action against it as production-impacting.

## Backup target (Dell appliance)

- Backups go to a **Dell PowerProtect DD** appliance, integrated via
  Veritas/Cohesity. Treat it as the snapshot storage target for Ops Manager backup.

## Test scenarios (each becomes its own guide)

Guides use two-digit ordered file name prefixes (see documentation-style guide).

0. **Install Ops Manager** (test mode, single host) — `00-install-ops-manager.md` (exists).
1. **Provision MongoDB on empty nodes** — Ops Manager automation deploys a MongoDB
   deployment onto empty nodes (Project A, new) — `10-provision-new.md`.
2. **Automation change with no downtime** — use Ops Manager to change a running
   parameter (e.g. the slow-query threshold / `slowOpThresholdMs`) and show the
   rolling change happens with no downtime.
3. **Monitoring & Performance Advisor on the production system** — attach Ops Manager
   to the live production replica set (Project B), monitor metrics, and use
   Performance Advisor on the real, existing production traffic.
   Do NOT load dummy data or inject synthetic slow queries against production.
   Performance Advisor should analyze the real workload already running. If synthetic
   slow-query generation is needed, it must be done on a separate staging/test
   deployment, never against production.

   3a. **Monitor the new replica set and test alerting** — `20-monitoring-new-and-alerting.md`.
   Watch metrics for `new-rs0` (Project A, test/lab), then create an alert with a low
   threshold, deliver it to a Telegram bot via an Ops Manager webhook (through a small
   relay, since the webhook payload is not Telegram's format), and trigger it by
   generating light activity. Runs on Project A so synthetic activity is safe.
4. **Backup and restore with the Dell appliance** — `45-backup-restore-third-party.md`. Uses
   Ops Manager's third-party backup integration (Veritas/Cohesity to Dell PowerProtect
   DD), since the appliance is integrated via Veritas/Cohesity, not exposed as native
   Ops Manager storage. Covers the full cycle: back up the live production replica set
   (Project B) to the appliance, then restore a snapshot into the `new-rs0` replica set
   (Project A). The restore overwrites `new-rs0`. Initial backup sync adds load;
   schedule during a low-traffic window. Requires Ops Manager 8.0.8+.
5. **Native Ops Manager backup and restore** — `40-native-backup.md`. Optional scenario
   using Ops Manager's own native backup instead of the third-party integration. Uses a
   file system snapshot store (local on `opsmgr-1`) plus a local oplog store, and backs
   up/restores the `new-rs0` replica set (Project A, test/lab). No external platform.
6. **Database-level audit log** — `50-database-audit-log.md`. Enable MongoDB Enterprise
   database auditing to see who read/wrote what at the DB operation level. This is
   auditing of database operations, NOT auditing of Ops Manager itself. Requires
   Enterprise binaries.
7. **Uninstall Ops Manager safely** — `60-uninstall-ops-manager.md`. Tear down the test
   Ops Manager environment while keeping the production replica set (Project B) running.
   Release production from management first (verify it stands alone, hand `mongod` back
   to systemd, relocate the keyfile), then remove agents and Ops Manager. Touches
   production, so use a change window.

## Guardrails

- Project A (new) is test/lab. Scenarios 0, 1, 2, 5, 6 and the restore half of scenario 4
  run against it.
- Project B is the REAL production replica set. Scenario 3 and the backup half of
  scenario 4 touch live production. For those production-facing parts:
  - Never write dummy/synthetic data or inject slow queries into production. Analyze
    only the real existing workload.
  - Attaching Ops Manager (installing the MongoDB Agent) and enabling backup are real
    changes to production hosts. Guides must call for a change window and approval,
    and prefer read-only/monitoring-only steps before any change.
  - Flag any step that adds load, changes configuration, or restarts a process on
    production, and recommend scheduling during low-traffic windows.
- Do not include real credentials, hostnames, or secrets. Use placeholders.
