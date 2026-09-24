# MongoDB Ops Manager Testing Guides

A collection of step-by-step guides for setting up and testing MongoDB Ops Manager, written so you can follow them end to end without hopping between MongoDB documentation pages.

## Guides

Follow them in order; the numbers indicate the intended sequence.

| # | Guide | What it covers |
|---|-------|----------------|
| 00 | [Install Ops Manager](00-install-ops-manager.md) | Install Ops Manager and its Application Database on a single test host, including infrastructure (VMs, ports, FQDN, NTP). |
| 10 | [Provision a new replica set](10-provision-new.md) | Use Ops Manager automation to deploy a brand-new MongoDB replica set onto empty nodes (Project A). |
| 20 | [Monitor new & test alerting](20-monitoring-new-and-alerting.md) | Enable monitoring on the new replica set, watch metrics, and configure/trigger an alert that notifies a Telegram bot via webhook. |
| 25 | [Monitor existing & Performance Advisor](25-monitoring-existing.md) | Attach Ops Manager to an existing production replica set (Project B) for monitoring and Performance Advisor. |
| 30 | [Change a parameter, no downtime](30-automation-no-downtime.md) | Change a running parameter (slow-query threshold) via automation and prove there is no downtime. |
| 40 | [Native backup & restore](40-native-backup.md) | Back up and restore using Ops Manager's own native backup (file system store), no third-party platform. |
| 45 | [Backup & restore (third-party / Dell)](45-backup-restore-third-party.md) | Back up production to a Dell PowerProtect DD appliance via Veritas/Cohesity, then restore into the new replica set. |
| 50 | [Database audit log](50-database-audit-log.md) | Enable MongoDB Enterprise database auditing to see who read/wrote what at the operation level. |
| 60 | [Uninstall Ops Manager (safe)](60-uninstall-ops-manager.md) | Tear down the test environment while keeping the production replica set running. |

## Environment at a glance

- **Ops Manager** 8.0 (8.0.8+ for third-party backup)
- **MongoDB** 8.0 Enterprise
- **OS** RHEL 9.4
- A single test Ops Manager host (`opsmgr-1`) with two projects: **Project A** (new, test/lab) and **Project B** (existing production).

## Reading the guides

The guides are plain markdown — open any `.md` file in a text editor or on your Git host.

For a nicer, rendered view in a browser, this project includes `index.html`, which lists
all guides in a sidebar and renders the markdown. Because it loads the files with
`fetch`, serve the folder over HTTP rather than opening the file directly:

```bash
python3 -m http.server 8000
```

Then open `http://localhost:8000/` in your browser. (Any static file server works; for
example `npx serve` if you prefer Node.)

## Notes

- Guides use placeholders in `<angle-brackets>` for values you supply (hostnames, keys, versions). Replace them with your own.
- Scenarios 3 and the backup half of scenario 4 touch real production — run those in an approved change window.
