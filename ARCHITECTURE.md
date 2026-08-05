# Velero Backup Controller Architecture

Complete reference for how Velero processes a Backup CR, from creation to completion. Covers the controller chain, volume protection decision tree, and concurrent activity timeline.

## Controller Chain

Four independent controllers watch the Backup CR and hand off via phase transitions. Each controller writes a phase to the CR status, which triggers the next controller's watch predicate.

```
Queue Controller → Backup Controller → [Operations Controller] → Finalizer Controller
```

The Operations Controller is only involved when async plugin operations (e.g., CSI data movers) are in progress.

### Queue Controller

**File:** `pkg/controller/backup_queue_controller.go`
**Watches:** Phase `""` (empty), `New`, `Queued`

| Step | Action | Lines |
|------|--------|-------|
| 1 | Predicate fires on Create/Update when phase is `""` or `New` | 104-119 |
| 2 | List all backups, find max QueuePosition | 248-258 |
| 3 | Set phase → `Queued`, QueuePosition = max + 1 | 259-261 |
| 4 | Check `RunningCount() >= concurrentBackups` → stay Queued if at limit | 274-277 |
| 5 | Check `detectNamespaceConflict()` with earlier backups → stay Queued if conflict | 279-287 |
| 6 | Check `checkForEarlierRunnableBackups()` → stay Queued if earlier backup can run | 288-292 |
| 7 | Set phase → `ReadyToStart`, QueuePosition → 0 | 293-295 |
| 8 | Call `backupTracker.AddReadyToStart()` | 296 |
| 9 | Re-number queue positions for remaining queued backups | 302-314 |

**Handoff:** Writes `ReadyToStart` to CR → Backup Controller's Update predicate fires.

**Re-check mechanism:** Queued backups are re-evaluated every 1 minute (PeriodicalEnqueueSource) and whenever any backup transitions out of InProgress.

### Backup Controller

**Files:** `pkg/controller/backup_controller.go`, `pkg/backup/backup.go`, `pkg/backup/item_backupper.go`
**Watches:** Phase `ReadyToStart` (Update events only)

| Step | Action | Lines |
|------|--------|-------|
| 1 | Update predicate fires on phase == ReadyToStart | 178-200 |
| 2 | Fetch CR, double-check phase (race guard against multiple replicas) | 278-306 |
| 3 | `prepareBackupRequest()`: validate BSL, VSLs, resource filters, set defaults | 395-635 |
| 4 | If validation errors → phase `FailedValidation` (terminal) | 312-313 |
| 5 | Otherwise → phase `InProgress`, set StartTimestamp | 315-316 |
| 6 | `backupTracker.Add()` | 329 |
| 7 | `runBackup()`: create plugins, temp files, run engine, persist artifacts | 746-891 |
| 8 | Determine result phase based on errors + async ops (see below) | 846-861 |
| 9 | Patch CR with result phase | 381-393 |

**Validation (prepareBackupRequest):**
- Resolves BSL (server default or user-specified), validates exists/not-read-only/available
- Validates VSLs: one per provider, fills defaults
- Sets defaults: TTL, backup format version, CSI timeout, item operation timeout
- Remaps deprecated `DefaultVolumesToRestic` → `DefaultVolumesToFsBackup`
- Excludes namespaces with `velero.io/exclude-from-backup=true` label
- Validates resource filters (old vs new API cannot be mixed)
- Auto-excludes CSI VolumeSnapshot/VolumeSnapshotContent resources
- Validates label selectors (cannot have both LabelSelector and OrLabelSelectors)
- Resolves resource policies from ConfigMap

**Result phase determination (`backup_controller.go:846-861`):**

| Condition | Phase |
|-----------|-------|
| Fatal errors | `Failed` (terminal) |
| Logged errors + in-progress async ops | `WaitingForPluginOperationsPartiallyFailed` |
| Logged errors + no async ops | `FinalizingPartiallyFailed` |
| No errors + in-progress async ops | `WaitingForPluginOperations` |
| No errors + no async ops | `Finalizing` |

**Handoff:** Writes result phase to CR → Operations Controller or Finalizer Controller picks up.

### Operations Controller

**File:** `pkg/controller/backup_operations_controller.go`
**Watches:** Phase `WaitingForPluginOperations`, `WaitingForPluginOperationsPartiallyFailed`
**Mechanism:** Periodic poller (every 10 seconds), not event-driven.

| Step | Action | Lines |
|------|--------|-------|
| 1 | Periodic reconciler fires every 10 seconds | 49 |
| 2 | Fetch backup, verify phase | 130-136 |
| 3 | Get BSL and backup store | 138-166 |
| 4 | Download operations list from object storage | 175-182 |
| 5 | For each in-progress operation: query plugin.Progress(operationID) | 289-386 |
| 6 | If plugin error → mark operation Failed | 303-309 |
| 7 | If completed with error → mark Failed | 310-315 |
| 8 | If completed successfully → mark Completed | 316-320 |
| 9 | If past ItemOperationTimeout → Cancel(), mark Failed | 363-371 |
| 10 | If any errors → phase `WaitingForPluginOperationsPartiallyFailed` | 198 |
| 11 | If no more in-progress ops: transition phase | 205-213 |

**Phase transitions:**
- `WaitingForPluginOperations` → `Finalizing`
- `WaitingForPluginOperationsPartiallyFailed` → `FinalizingPartiallyFailed`

**Handoff:** Writes `Finalizing*` to CR → Finalizer Controller's watch predicate fires.

### Finalizer Controller

**File:** `pkg/controller/backup_finalizer_controller.go`
**Watches:** Phase `Finalizing`, `FinalizingPartiallyFailed`

| Step | Action | Lines |
|------|--------|-------|
| 1 | Watch predicate fires on phase Finalizing or FinalizingPartiallyFailed | 108-114 |
| 2 | Get BSL and backup store | 137-151 |
| 3 | Download item operations list | 154-158 |
| 4 | If operations exist: download backup tarball, call `FinalizeBackup()` | 167-203 |
| 5 | FinalizeBackup() processes PostOperationItems from async ops | 192 |
| 6 | Re-runs BIA plugins on modified items, builds new tarball | 192 |
| 7 | Finalizing → `Completed`; FinalizingPartiallyFailed → `PartiallyFailed` | 205-214 |
| 8 | Set CompletionTimestamp | 216 |
| 9 | Upload updated metadata and tarball | 222-235 |
| 10 | `backupTracker.Delete()` | 117-135 |

### Phase State Machine

```
"" / New
  → Queued                          [Queue Controller]
    → ReadyToStart                  [Queue Controller]
      → FailedValidation            [Backup Controller — terminal]
      → InProgress                  [Backup Controller]
        → Failed                    [Backup Controller — terminal]
        → WaitingForPluginOps       [Backup Controller → Operations Controller]
          → WaitingPartiallyFailed  [Operations Controller]
            → FinalizingPartiallyFailed → PartiallyFailed [Finalizer — terminal]
          → Finalizing              [Operations Controller → Finalizer Controller]
            → Completed             [Finalizer — terminal]
        → FinalizingPartiallyFailed [Backup Controller → Finalizer Controller]
          → PartiallyFailed         [Finalizer — terminal]
        → Finalizing                [Backup Controller → Finalizer Controller]
          → Completed               [Finalizer — terminal]
```

Terminal phases: `Completed`, `PartiallyFailed`, `Failed`, `FailedValidation`

### Backup Tracker

**File:** `pkg/controller/backup_tracker.go`

Tracks backup lifecycle for concurrency management:
- `AddReadyToStart()` — counted in RunningCount()
- `Add()` — InProgress, counted in RunningCount()
- `AddPostProcessing()` — WaitingForPluginOps/Finalizing, NOT counted in RunningCount()
- `Delete()` — terminal phase, removed from all tracking
- `RunningCount()` = readyToStartBackups.Len() + inProgressBackups.Len()

## Backup Engine Internals

### BackupWithResolvers (`backup/backup.go:272-756`)

1. Create gzip/tar writers
2. Write backup version file
3. Resolve namespace includes/excludes (wildcard expansion, ArgoCD-managed detection)
4. Resolve resource includes/excludes
5. Parse BackupResourceHookSpecs → ResourceHook objects
6. Resolve BIA and ItemBlock actions from plugins
7. Set up pod volume backupper with timeout context
8. **ItemCollector** queries K8s API for all matching resources
9. Build PVC-to-Pod cache for volume policy lookups
10. Create itemBackupper engine
11. **Start progress updater goroutine** (patches CR every 1 second)
12. Group items into BackupItemBlocks
13. Execute ItemBlockActions to find related items per block
14. Send blocks to worker pool for concurrent processing
15. Wait for all pod volume backups
16. Back up associated CRDs

### ItemBlock Processing (`backup/backup.go:854-899`)

Each block is processed by a worker:
1. Identify pods in the block
2. Run pre-hooks on pods (`handleItemBlockPreHooks`)
3. Back up each item via `backupItem()`
4. Run post-hooks on pods (`handleItemBlockPostHooks`)
5. Post-hooks wait for PodVolumeBackups to complete first

### Per-Item Backup (`backup/item_backupper.go:190-360`)

1. **Inclusion checks:** skip if exclude-from-backup label, namespace excluded, resource excluded, deletion timestamp set
2. **Dedup:** skip if already backed up
3. **Pod volume handling:** for Pods, call `ShouldPerformFSBackup()` per volume, track in podVolumeSnapshotTracker
4. **Execute BackupItemActions:** plugins can modify item, add related items, start async ops
5. **PV snapshot:** for PersistentVolumes, call `takePVSnapshot()`
6. **Pod volumes:** for Pods, call `backupPodVolumes()`
7. **Serialize:** marshal to JSON, write to tar archive

## Volume Protection Decision Tree

When a volume is encountered during backup, the following decision tree determines how it is protected. Volume policies have highest priority, followed by CSI, then native snapshots, then FSB via annotations.

### Priority Order

1. **Volume Policy** (from ResourcePolicy ConfigMap) — if matched, overrides everything
2. **FSB via Pod processing** — runs first during Pod backup, tracks volumes to prevent duplicate snapshot
3. **CSI Snapshot via BIA** — runs during PVC backup if EnableCSI=true
4. **Native VolumeSnapshotter** — runs during PV backup for non-CSI volumes

### Decision Tree

```
Is a ResourcePolicy ConfigMap configured with volumePolicies?
├── YES: Does a policy rule match this volume?
│   ├── YES: What action?
│   │   ├── fs-backup → KOPIA FILE SYSTEM BACKUP
│   │   ├── snapshot  → CSI or NATIVE SNAPSHOT (depends on PV type)
│   │   ├── skip      → NO PROTECTION
│   │   └── custom    → EXTERNAL PLUGIN
│   └── NO: Fall through to legacy logic (below)
└── NO: Fall through to legacy logic

Legacy logic (no policy match):

What resource type is being processed?

PODS (FSB path, runs first):
  Is DefaultVolumesToFsBackup = true?
  ├── YES (opt-out mode):
  │   Is volume in exclude annotation (backup.velero.io/backup-volumes-excludes)?
  │   ├── YES → SKIP FSB (may still get snapshot via PVC/PV path)
  │   └── NO: Is volume type eligible? (excludes HostPath, Secret, ConfigMap, Projected, DownwardAPI)
  │       ├── YES → KOPIA FILE SYSTEM BACKUP
  │       └── NO  → SKIP (ineligible type)
  └── NO (opt-in mode, default):
      Is volume in opt-in annotation (backup.velero.io/backup-volumes)?
      ├── YES → KOPIA FILE SYSTEM BACKUP
      └── NO  → NO FSB (may still get snapshot via PVC/PV path)

PVCs (CSI path):
  Is EnableCSI feature flag = true?
  ├── NO  → CSI DISABLED (no VolumeSnapshot created)
  └── YES: Is the PV CSI-backed? (pv.Spec.CSI != nil)
      ├── NO  → NOT CSI (falls through to native snapshot on PV)
      └── YES: Is SnapshotMoveData = true?
          ├── YES → CSI SNAPSHOT + KOPIA DATA MOVER (async DataUpload)
          └── NO  → CSI VOLUMESNAPSHOT IN-CLUSTER (VS/VSC in tar)

PVs (Native snapshot path):
  Is PV CSI-backed and EnableCSI = true?
  ├── YES → SKIPPED (handled by CSI BIA on the PVC)
  └── NO: Is SnapshotVolumes != false and a VolumeSnapshotter recognizes the PV?
      ├── YES → NATIVE CLOUD SNAPSHOT (AWS EBS / GCP PD / Azure Disk)
      └── NO  → NO PROTECTION
```

### Volume Protection Methods

#### Kopia File System Backup
- **Trigger:** Pod annotation opt-in, DefaultVolumesToFsBackup=true, or volume policy action=fs-backup
- **Mechanism:** PodVolumeBackup CR created per volume. Node-agent DaemonSet on the pod's node runs Kopia uploader to copy files to BSL Kopia repository.
- **Files:** `pkg/podvolume/backupper.go:233-390`, `internal/volumehelper/volume_policy_helper.go:221-291`
- **Repository type:** Always Kopia (Restic fully removed). Hardcoded at `pkg/podvolume/util.go:174`.
- **Async:** No — post-hooks block until PVBs complete.
- **Duplicate prevention:** Volumes tracked in podVolumeSnapshotTracker, which causes ShouldPerformSnapshot() to return false for the corresponding PV.

#### CSI VolumeSnapshot (In-Cluster)
- **Trigger:** EnableCSI=true, PV is CSI-backed, SnapshotMoveData=false
- **Mechanism:** CSI BIA plugin (`velero.io/csi-pvc-backupper`) creates VolumeSnapshot CR. CSI driver creates the snapshot. VS + VolumeSnapshotContent included in backup tar.
- **Files:** `pkg/backup/actions/csi/pvc_action.go:282-459`
- **Async:** No — Velero blocks waiting for VSC readyToUse=true (up to CSISnapshotTimeout).
- **Restore:** VSC used as pre-provisioned data source for new PVC. NOT portable across clusters.

#### CSI Snapshot + Kopia Data Mover
- **Trigger:** EnableCSI=true, PV is CSI-backed, SnapshotMoveData=true
- **Mechanism:** CSI BIA creates VolumeSnapshot, then creates DataUpload CR. Node-agent mounts snapshot, runs Kopia uploader to stream data to BSL. VolumeSnapshot is transient (deleted after upload).
- **Files:** `pkg/backup/actions/csi/pvc_action.go:375-431`
- **Async:** YES — DataUpload tracked via operationID. Operations controller polls until complete. Backup phase = WaitingForPluginOperations.
- **Restore:** DataDownload CR re-hydrates volume from BSL. Portable across clusters.

#### Native Cloud Snapshot
- **Trigger:** Non-CSI PV (or CSI disabled), VolumeSnapshotter plugin recognizes the PV
- **Mechanism:** `takePVSnapshot()` iterates VolumeSnapshotLocations, calls each provider's GetVolumeID(). First match calls CreateSnapshot() via cloud API.
- **Files:** `pkg/backup/item_backupper.go:577-732`
- **Providers:** AWS EBS, GCP PD, Azure Disk (external VolumeSnapshotter plugins).
- **Async:** No — CreateSnapshot() is synchronous (though the cloud snapshot itself may be eventually consistent).
- **Restore:** CreateVolumeFromSnapshot() creates new volume, SetVolumeID() updates PV.

### Key Configuration Fields

| Field | Type | Default | Effect |
|-------|------|---------|--------|
| `spec.snapshotVolumes` | *bool | nil (true) | Enable/disable all volume snapshots |
| `spec.defaultVolumesToFsBackup` | *bool | nil (false) | false=opt-in FSB via annotation; true=opt-out via annotation |
| `spec.snapshotMoveData` | *bool | nil (false) | true=upload CSI snapshot data to BSL via Kopia |
| `spec.resourcePolicy` | string | "" | ConfigMap name with volume/resource policies |
| `spec.csiSnapshotTimeout` | Duration | 10m | Timeout waiting for CSI snapshot readiness |
| `spec.itemOperationTimeout` | Duration | 4h | Timeout for async plugin operations (DataUploads) |
| `spec.dataMover` | string | "" | Data mover identifier ("" or "velero" = built-in) |

### Key Annotations

| Annotation | On | Effect |
|------------|-----|--------|
| `backup.velero.io/backup-volumes` | Pod | Comma-separated volume names to opt IN to FSB |
| `backup.velero.io/backup-volumes-excludes` | Pod | Comma-separated volume names to opt OUT of FSB |
| `velero.io/exclude-from-backup` | Any (label) | If "true", resource excluded from backup entirely |

### Volume Policy Actions

Defined in `internal/resourcepolicies/resource_policies.go:40-51`:

| Action | Effect |
|--------|--------|
| `skip` | No protection at all |
| `fs-backup` | Force Kopia file-system backup |
| `snapshot` | Force CSI or native snapshot |
| `custom` | External plugin handles it |

Policy match conditions (all must match, first matching policy wins):
- `capacity` — volume size range (e.g., "10Gi,100Gi")
- `storageClass` — storage class name(s)
- `csi.driver` — CSI driver name
- `nfs` — NFS server/path
- `volumeTypes` — volume type (local, hostPath, etc.)
- `pvcLabels` — PVC label matching
- `pvcPhase` — PVC phase (Pending, Bound, Lost)
- `pvcVolumeMode` — Filesystem or Block
- `pvcAccessModes` — ReadWriteOnce, ReadOnlyMany, ReadWriteMany

## Concurrent Activity During Backup

### What Runs in Parallel

During a backup, multiple goroutines and external processes run concurrently:

| Activity | Lifetime | Runs on |
|----------|----------|---------|
| Progress updater goroutine | Entire BackupWithResolvers() duration | Velero server |
| ItemBlock worker pool | After item collection until all blocks processed | Velero server |
| Pre/post hooks per block | Per block, within worker | Velero server → target pod |
| PodVolumeBackup (Kopia) | After PVB CR created until upload complete | Node-agent on pod's node |
| CSI driver snapshot | After VolumeSnapshot CR created until readyToUse | CSI driver controller |
| DataUpload (Kopia) | After DataUpload CR created until upload complete | Node-agent on node with snapshot |
| Operations controller polling | After backup engine completes, every 10s | Velero server |

### Dependency Chain

```
Item Collection
  └──blocks──→ Worker Pool (concurrent blocks)
                 ├── Pre-hooks
                 │     └──triggers──→ PVB CRs created
                 │                      ├── Node-agent (node-1) ──parallel──┐
                 │                      └── Node-agent (node-2) ──parallel──┤
                 │                                                          │
                 │     Post-hooks ←──blocks──────────────────────────────────┘
                 │       (waits for ALL PVBs before running post-hooks)
                 │
                 ├── CSI BIA on PVC
                 │     └──triggers──→ CSI Driver creates snapshot
                 │                      └──blocks──→ VolumeSnapshot ready
                 │                                     ├── SnapshotMoveData=false: include VS in tar
                 │                                     └── SnapshotMoveData=true: create DataUpload CR
                 │                                           └──triggers──→ Node-agent Kopia upload (ASYNC)
                 │
                 └── takePVSnapshot() for PVs
                       └── VolumeSnapshotter.CreateSnapshot() (synchronous cloud API)

BackupWithResolvers() completes
  └──triggers──→ Persist artifacts to BSL
                   └──triggers──→ Phase write (Finalizing or WaitingForPluginOps)

If WaitingForPluginOps:
  Operations Controller polls every 10s
    └──blocks──→ All DataUploads complete
                   └──triggers──→ Phase = Finalizing

Finalizer Controller picks up Finalizing
  └── Download tarball, FinalizeBackup(), upload
        └──triggers──→ Phase = Completed
```

### What Blocks What

| Blocked Activity | Waits For | Why |
|-----------------|-----------|-----|
| Worker pool start | Item collection complete | Can't process items that haven't been discovered |
| Post-hooks on a block | ALL PVBs in that block complete | Ensures data consistency (e.g., fsthaw after all copies done) |
| VS inclusion in tar | CSI driver reports readyToUse=true | Can't back up a snapshot that doesn't exist yet |
| Persist to BSL | BackupWithResolvers() returns | Need the complete tar before uploading |
| Finalizer controller | Operations controller sets Finalizing | Can't finalize while async ops are running |
| Operations controller done | ALL DataUploads complete/failed/timeout | Must wait for every async operation |

### What Does NOT Block

| Activity A | Activity B | Why Independent |
|-----------|-----------|----------------|
| Node-agent on node-1 | Node-agent on node-2 | Different nodes, different DaemonSet pods |
| Progress updater goroutine | Worker pool | Progress updater reads counters, doesn't modify state |
| CSI driver snapshot | Other item processing | Snapshot creation is async to the CSI driver |
| DataUpload Kopia stream | Backup controller | DataUpload runs AFTER backup controller is done (different controller) |
