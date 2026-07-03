# Velero design and failure modes (+ etcd for self-hosted)

## Core layout

- **BackupStorageLocation (BSL)** — the object-storage bucket for backup metadata + resource
  manifests. Must be off-cluster (losing the cluster and its backups together is the failure
  this whole domain exists to prevent). One `default` BSL; extra BSLs for cross-region copies.
- **Schedules** — cron-shaped `Schedule` objects with `ttl` (default 30d). Typical shape:
  frequent (hourly/6h) short-TTL for the tight-RPO namespaces + daily cluster-wide with longer
  TTL. Scope with `includedNamespaces`/label selectors — backing up everything hourly is how
  buckets and backup windows blow up.
- **What a backup contains**: resource manifests (incl. **Secrets in plaintext** unless
  configured otherwise — the bucket's IAM/encryption is part of the design) + volume data via
  one of the two paths below.

## Volume data: CSI snapshots vs file-system backup (node-agent/kopia)

| Dimension | CSI snapshots (+ data mover) | File-system backup (FSB) |
|---|---|---|
| Mechanism | VolumeSnapshot via the CSI driver; data mover copies snapshot content to the BSL bucket | node-agent pod reads the live filesystem, uploads with kopia |
| Consistency | Crash-consistent at an instant (snapshot semantics) | File-by-file over time — inconsistent for live databases *by construction* |
| Requirements | CSI driver with snapshot support + `VolumeSnapshotClass` labeled for Velero (`velero.io/csi-volumesnapshot-class`) | Any volume a pod mounts (incl. NFS/local-path — the homelab path) |
| Restore speed | Fast within the same backend; cross-cluster needs the data-mover copy | Full data re-upload/download always |
| Choose when | Backend has real snapshot support (cloud disks, Ceph, Longhorn) | No snapshot support, or portability across backends matters |

Opt-in/opt-out matters: FSB is per-pod-volume (annotation `backup.velero.io/backup-volumes`
opt-in, or `--default-volumes-to-fs-backup` opt-out) — an unannotated volume in opt-in mode is
**silently not backed up**; reviews check which mode is active and what falls through.

## Consistency hooks (databases)

Crash-consistent is not application-consistent. For database-class workloads either:

- **Pre/post hooks** — `pre.hook.backup.velero.io/command` (e.g. `fsfreeze`, `pg_backup_start`,
  a flush) and matching post hook; hook failure fails the backup **only if** `onError: Fail` —
  the default `Continue` produces "successful" inconsistent backups.
- **Or the DB's own tooling** — an operator with backup support (CloudNativePG, etc.) or dump
  jobs shipping to the same bucket; Velero then covers manifests while the operator owns data.
  State which tool owns data-consistency per database — "both, vaguely" means neither.

## Restore semantics (where restores go wrong)

- `existingResourcePolicy: none` (default) **skips** resources that already exist — a restore
  into a half-broken namespace quietly restores nothing; `update` patches toward the backup.
  Say which is intended.
- Namespace remap (`--namespace-mappings`) restores into a scratch namespace — the safe default
  for restore *tests* and partial recoveries.
- PV restore: with CSI, volumes re-provision from snapshot; with FSB, data re-uploads into
  fresh PVs — both change PV identity (new volume handles); apps caring about volume identity
  need attention.
- Ordering: Velero restores CRDs/namespaces first, but operator-managed apps may need the
  operator running before its CRs restore cleanly — restore-test failures here are design
  findings, not restore bugs.
- `velero restore describe --details` names every skipped/failed item — read it; a `Completed`
  restore with 40 warnings is not a success.

## Failure modes

| Symptom | Cause |
|---|---|
| Backup `PartiallyFailed` | Usually per-volume: snapshot class missing/unlabeled, node-agent down on one node, hook failed with `Continue` — `velero backup describe --details` + `velero backup logs` name the item |
| Backup `Failed` immediately | BSL unreachable (credentials, bucket policy, network) — `velero backup-location get` shows `Unavailable` |
| Schedules stopped producing backups | Velero pod down/CRD conflict after upgrade; last-success age is the metric to alert on (`velero_backup_last_successful_timestamp`) |
| Restore hangs `InProgress` | Data mover/node-agent stuck; huge FSB volumes just take long — check node-agent logs before killing anything |
| Restored app starts but data is stale/corrupt | Crash-consistent backup of a DB without hooks — the consistency section above |

## etcd layer (self-hosted control planes only)

- `etcdctl snapshot save` on a schedule (systemd timer / CronJob on a control-plane node),
  copied **off the node**, plus the PKI dir (`/etc/kubernetes/pki`) — a snapshot without the
  CA certs restores a cluster nobody can talk to.
- Verify snapshots (`etcdutl snapshot status`) as part of the schedule, not at restore time.
- Restore (`etcdutl snapshot restore` + static-pod surgery) is the most destructive operation
  in Kubernetes — it rewinds *everything* the API server knows, including things Velero would
  have restored selectively. In a review it appears as a documented, plan-only, last-resort
  procedure with its blast radius stated; kubeadm/k3s each have their own documented variant
  (k3s: built-in `etcd-snapshot` with S3 support — use it rather than hand-rolling).
- Managed control planes (EKS/GKE/AKS) delete this whole section — but nothing else.
