# Workload Kind, Storage & API-Drift Review Reference

Distilled from LukasNiessen/kubernetes-skill (`references/deployment-patterns.md`, `stateful-patterns.md`, `job-patterns.md`, `daemonset-operator-patterns.md`, `storage-and-state.md`, `api-drift.md`) during third-party review.

## Choosing the workload kind

| Need | Kind | Red flag if used instead |
|---|---|---|
| Stateless, interchangeable pods (APIs, workers, proxies) | Deployment | StatefulSet "just in case" — extra complexity, no benefit |
| Stable per-pod identity + per-pod storage (Postgres, Kafka, etcd) | StatefulSet | Deployment sharing one RWO PVC across replicas (multi-attach failure) |
| Finite run-to-completion (migrations, ETL) | Job | Deployment that exits → CrashLoopBackOff |
| Recurring scheduled work | CronJob | while-true loop inside a Deployment |
| Exactly one pod per (qualifying) node (log/metric agents, CNI, CSI) | DaemonSet | Deployment with replicas ≈ node count |
| Long-running queue consumer | Deployment + HPA | Job — never completes |
| Storage but no per-pod identity | Deployment + single PVC | StatefulSet |

Deployment production floor: `replicas >= 2`, requests AND limits, PSS-restricted securityContext, readiness+liveness on separate dedicated `/healthz/*` endpoints, `topologySpreadConstraints` (zone), PDB. Never put `app.kubernetes.io/version` in `selector.matchLabels` — selectors are immutable and this breaks every upgrade. HPA: set `behavior.scaleDown.stabilizationWindowSeconds: 300` (default scale-down thrashes).

## StatefulSet review points

- **Headless Service required**: `clusterIP: None`, and `spec.serviceName` must match it exactly (omitting it is an API error). Per-pod DNS: `<pod>.<headless-svc>.<ns>.svc.cluster.local`.
- **`volumeClaimTemplates` is a peer of `template`**, not nested under `template.spec` (frequent generation error). It is also effectively immutable — you cannot change size/class of existing per-pod PVCs through the template; new values apply only to future ordinals.
- **PVCs survive scale-down and StatefulSet deletion by design.** Cleanup is manual, or via `persistentVolumeClaimRetentionPolicy` (1.27+).
- **`podManagementPolicy`**: `OrderedReady` (default, sequential, each Ready before next — consensus systems) vs `Parallel` (independent init, e.g. Cassandra).
- **`updateStrategy`**: `RollingUpdate` proceeds in reverse ordinal order; `rollingUpdate.partition: N` = canary (only ordinals >= N update). `OnDelete` = manual pod-by-pod control for careful DB upgrade sequencing.
- **Access mode**: `ReadWriteOnce` for databases; `ReadWriteOncePod` for strict single-pod attach. ❌ RWX for single-node DBs.
- **`terminationGracePeriodSeconds`**: default 30s is too short for databases — expect 60–120s.
- **PDB with `maxUnavailable: 1`** on every stateful workload.
- Postgres-specific: `PGDATA` must point to a subdirectory of the mount (mount root may contain `lost+found`).

## Job / CronJob correctness

| Field | Rule |
|---|---|
| `restartPolicy` | Must be `Never` or `OnFailure` — the pod default `Always` is rejected |
| `backoffLimit` | Default 6; lower to 2–3 for non-transient failures |
| `activeDeadlineSeconds` | Always set — unbounded stuck Jobs run forever |
| `ttlSecondsAfterFinished` | Always set (3600 is a safe default) — otherwise finished Jobs+pods accumulate indefinitely; ship logs externally first |
| `completions`/`parallelism` | Default 1/1; `completionMode: Indexed` gives `JOB_COMPLETION_INDEX` env — pointless if the container never reads it |
| `concurrencyPolicy` (CronJob) | **`Forbid` by default**; `Allow` only for independent runs; `Replace` when only the latest matters |
| `startingDeadlineSeconds` | Set (e.g. 600) to skip stale runs after controller downtime instead of bursting |
| `timeZone` (1.27+) | Set explicitly; otherwise schedule uses controller clock (usually UTC) |
| history limits | `successfulJobsHistoryLimit` / `failedJobsHistoryLimit` explicit |

**Idempotency is mandatory**: any `backoffLimit > 0` means at-least-once execution — upserts not inserts, check for completed work, unique output paths.

`podFailurePolicy` (1.26+) distinguishes failure classes:

```yaml
podFailurePolicy:
  rules:
    - action: FailJob                       # fatal exit code — don't retry
      onExitCodes: {containerName: worker, operator: In, values: [1]}
    - action: Ignore                        # node drain — free retry
      onPodConditions: [{type: DisruptionTarget}]
```

CronJob has three label layers (CronJob, `jobTemplate`, pod template) — all three need consistent labels or selectors/monitoring miss the pods.

## DaemonSet review points

- No `replicas` field exists — including one is an API error.
- `updateStrategy`: `RollingUpdate` with `maxUnavailable: "10%"` for large clusters; `OnDelete` for critical infra (CNI, kube-proxy) needing manual pacing.
- Tolerations: add only specific keys (`node-role.kubernetes.io/control-plane` etc.). ❌ `operator: Exists` with no `key` — tolerates everything including NoExecute eviction taints.
- Requests multiply by node count: 200m × 100 nodes = 20 cores. Keep minimal, right-size from usage.
- PriorityClass: custom class in the 100000–10000000 range; `system-node-critical`/`system-cluster-critical` are reserved for core components.
- `hostPath` volumes need `type:` (Directory/File/Socket) to fail at mount time, not runtime; mount `readOnly: true` where possible.
- CRDs (operator patterns): always define `openAPIV3Schema` with required fields, min/max, enum, pattern — schemaless CRDs accept arbitrary YAML. Don't build an operator where Helm/Kustomize/Job suffices.

## Storage review points

| StorageClass field | Production value | Failure if wrong |
|---|---|---|
| `reclaimPolicy` | `Retain` | Default `Delete` destroys the cloud volume when the PVC goes |
| `volumeBindingMode` | `WaitForFirstConsumer` | Default `Immediate` can provision the volume in a different AZ than the pod → unschedulable |
| `allowVolumeExpansion` | `true` | Resize requires PVC delete/recreate (data loss under `Delete`) |
| `parameters.encrypted` | `"true"` | Unencrypted at rest |

- Access mode must match the driver: block storage (EBS `ebs.csi.aws.com`, GCE PD, Azure Disk) is RWO-only — an RWX PVC on it stays **Pending forever**. RWX needs file storage (EFS, Filestore, Azure Files, NFS/CephFS). `ReadWriteOncePod` for DBs to prevent multi-attach corruption.
- Deployment `replicas > 1` + RWO PVC = only one pod runs; use StatefulSet volumeClaimTemplates or RWX.
- `emptyDir` always needs `sizeLimit` (unbounded emptyDir can fill node disk and evict every pod on the node); `medium: Memory` counts against the memory limit.
- `fsGroup` in pod securityContext or the non-root user can't write the mounted volume.
- Expansion: patch PVC `spec.resources.requests.storage` upward only (never shrink); some CSI drivers need a pod restart for block volumes.
- Snapshots: `VolumeSnapshot` (+ `volumeSnapshotClassName`) before any destructive change; restore = new PVC with `dataSource: {kind: VolumeSnapshot}`. CSI snapshots can be crash-inconsistent — pair with logical backups (pg_dump/mysqldump) and test restores.

## API deprecation / drift checks

Deprecated = warning, still works. Removed = hard apply failure, no fallback. LLM/tutorial output frequently carries removed versions — verify every `apiVersion`, never trust memory.

| Resource | Current | Removed versions (in) |
|---|---|---|
| Ingress | `networking.k8s.io/v1` | `extensions/v1beta1`, `networking.k8s.io/v1beta1` (1.22) |
| PodDisruptionBudget | `policy/v1` | `policy/v1beta1` (1.25) |
| HorizontalPodAutoscaler | `autoscaling/v2` | `v2beta1` (1.25), `v2beta2` (1.26) |
| CronJob | `batch/v1` | `batch/v1beta1` (1.25) |
| EndpointSlice | `discovery.k8s.io/v1` | `v1beta1` (1.25) |
| CSIDriver/CSINode, CSR, TokenReview | `*/v1` | `*/v1beta1` (1.22) |
| FlowSchema/PriorityLevelConfiguration | `flowcontrol.apiserver.k8s.io/v1` | v1beta1 (1.26), v1beta2 (1.29), v1beta3 (1.32) |
| Workloads / core / RBAC / NetPol / SC | `apps/v1`, `v1`, `rbac.authorization.k8s.io/v1`, `networking.k8s.io/v1`, `storage.k8s.io/v1` | — |

Ingress v1 structural changes (all four must be checked together):
- `spec.backend` → `spec.defaultBackend`
- `serviceName`/`servicePort` → nested `service.name` / `service.port.number|name`
- `pathType` **required** on every path
- `kubernetes.io/ingress.class` annotation → `spec.ingressClassName`

HPA v2: `targetAverageUtilization` → `target.averageUtilization`; `behavior` is stable; `ContainerResource` for per-container scaling. PDB v1: `spec.selector` immutable after creation.

Validation (read-only):

```bash
pluto detect-files -d manifests/                 # deprecated/removed APIs in files
pluto detect-api-resources --cluster             # in a live cluster
kubeconform -kubernetes-version 1.29.0 -strict -summary manifests/   # pin to target version; -strict rejects unknown fields
helm template rel ./chart | kubeconform -kubernetes-version 1.29.0 -strict
kustomize build overlays/production | kubeconform -kubernetes-version 1.29.0 -strict
kubectl api-versions | grep networking.k8s.io    # what the cluster actually serves
kubectl get all -A -o json | jq -r '.items[] | select(.apiVersion | test("beta")) | .apiVersion + " " + .kind + " " + .metadata.namespace + "/" + .metadata.name'
```

Helm charts supporting multiple cluster versions should branch on `.Capabilities.APIVersions.Has "networking.k8s.io/v1"`. Kustomize: a patch `target` with wrong group/version silently matches nothing.
