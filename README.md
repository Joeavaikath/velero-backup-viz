# Velero Backup Controller Visualization

[![GitHub Pages](https://img.shields.io/badge/Live_Demo-GitHub_Pages-blue?logo=github)](https://joeavaikath.github.io/velero-backup-viz/)
[![Based on Velero](https://img.shields.io/badge/Based_on-Velero_main-blueviolet?logo=kubernetes)](https://github.com/vmware-tanzu/velero)
[![Static Site](https://img.shields.io/badge/Tech-Static_HTML%2FJS-orange)](#)
[![License](https://img.shields.io/badge/License-Apache_2.0-green)](LICENSE)

Interactive visualization of how Velero processes backup operations internally. Three views covering the controller chain, volume protection decision tree, and concurrent activity timeline.

**[Open the visualization](https://joeavaikath.github.io/velero-backup-viz/)**

Built from reading the Velero source at [`vmware-tanzu/velero@b74f8c9`](https://github.com/vmware-tanzu/velero/commit/b74f8c951).

## Views

### [Controller Flow](https://joeavaikath.github.io/velero-backup-viz/flow.html)

Swim-lane diagram showing how a Backup CR moves between Velero's four controllers:

- **Queue Controller** — concurrency limits, namespace conflict detection, FIFO ordering
- **Backup Controller** — validation, backup engine, item processing
- **Operations Controller** — polls async plugin operations (CSI data movers)
- **Finalizer Controller** — processes async results, writes terminal phase

Each controller's lane shows its specific actions. Handoff banners appear between lanes when the backup transitions from one controller to another via phase writes.

**Simulation paths:** Happy Path, Validation Failure, Partial Failure, Fatal Error, Kopia FSB, CSI Snapshot, CSI + Data Mover, Native Snapshot.

### [Volume Decision Tree](https://joeavaikath.github.io/velero-backup-viz/decision-tree.html)

Step-by-step wizard that walks through the volume protection decision logic. Answer questions about your configuration and it shows the exact code path:

- Volume policy resolution (ResourcePolicy ConfigMap)
- DefaultVolumesToFsBackup opt-in vs opt-out
- Pod annotation handling
- CSI snapshot with/without data mover
- Native cloud snapshots (AWS EBS, GCP PD, Azure Disk)

Five possible outcomes: **Kopia FSB**, **CSI VolumeSnapshot (in-cluster)**, **CSI + Kopia Data Mover**, **Native Cloud Snapshot**, or **No Protection**.

### [Timeline](https://joeavaikath.github.io/velero-backup-viz/timeline.html)

Gantt-style chart showing what runs in parallel during a backup:

- Dependency arrows (red = blocks on, orange = triggers)
- Annotated callouts at key moments (handoffs, blocking points, parallel activities)
- Click any bar to see its dependencies, what triggers it, and what runs alongside it

**Scenarios:** Happy Path (metadata only), Kopia FSB, CSI Snapshot, CSI + Data Mover, Native Snapshot.

## AI-Readable Reference

[`ARCHITECTURE.md`](ARCHITECTURE.md) contains the same information as structured prose — the complete controller chain, volume decision tree, dependency graph, and concurrency model. Designed to be consumed by AI agents, used as a CLAUDE.md reference, or read as documentation.

## Tech

Three static HTML files, zero dependencies. All styling and logic is inline. Runs from `file://` or any static host.

```
index.html          # Landing page — what is Velero, what the views cover
flow.html           # Controller flow (swim lanes + simulations)
decision-tree.html  # Volume protection wizard
timeline.html       # Concurrent activity Gantt chart
ARCHITECTURE.md     # Machine-readable reference
```

## Source

Based on reading Velero source at commit [`b74f8c951`](https://github.com/vmware-tanzu/velero/commit/b74f8c951). Key files traced:

| File | What it covers |
|------|---------------|
| `pkg/controller/backup_queue_controller.go` | Queue management, concurrency, namespace conflicts |
| `pkg/controller/backup_controller.go` | Validation, runBackup(), phase determination |
| `pkg/controller/backup_operations_controller.go` | Async operation polling |
| `pkg/controller/backup_finalizer_controller.go` | Finalization, async result processing |
| `pkg/backup/backup.go` | Core engine, item collection, block processing |
| `pkg/backup/item_backupper.go` | Per-item logic, BIA execution, PV snapshots |
| `pkg/backup/actions/csi/pvc_action.go` | CSI VolumeSnapshot, DataUpload creation |
| `pkg/podvolume/backupper.go` | PodVolumeBackup creation, Kopia uploader |
| `internal/volumehelper/volume_policy_helper.go` | ShouldPerformSnapshot/FSBackup decision logic |
| `internal/resourcepolicies/resource_policies.go` | Volume policy matching and actions |
