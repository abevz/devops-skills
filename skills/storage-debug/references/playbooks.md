# Storage event playbooks

Route by the event text from `kubectl describe pvc` / `kubectl describe pod`. All commands
read-only unless marked.

## PVC Pending (provisioning layer)

| Event / state | Cause | Diagnose / fix |
|---|---|---|
| No events, PVC Pending, SC has `volumeBindingMode: WaitForFirstConsumer` | **Normal** — binding waits for a pod to schedule | Not a bug; if a pod exists and it's still Pending, debug the *pod's* scheduling (it may be unschedulable for non-storage reasons) |
| `no persistent volumes available ... and no storage class is set` | PVC has no SC and no default SC exists | `kubectl get sc` — exactly one should carry the `is-default-class` annotation; zero or two both break class-less PVCs |
| `storageclass.storage.k8s.io "<name>" not found` | Typo'd/removed SC | Fix the claim or recreate the class — SC objects are config, safe to (re)apply |
| `ProvisioningFailed` with a backend error | CSI controller can reach the backend but create fails: quota/capacity full, credentials, wrong parameters | CSI controller pod logs quote the backend error verbatim — read them before theorizing |
| `ProvisioningFailed` `waiting for a volume to be created` repeating with no detail | CSI controller not running / webhook / RBAC broken | `kubectl get pods -n <csi-namespace>`; controller logs |
| Static PV expected but no bind | Capacity/accessModes/SC/selector mismatch between PV and PVC | Compare `kubectl get pv` fields against the claim — binding is exact-match on class and compatible on mode/size |

## ContainerCreating (attach/mount layer)

| Event | Cause | Diagnose / fix |
|---|---|---|
| `Multi-Attach error for volume` | RWO volume still attached to another node — usually after node failure or a stuck drain | See walkthrough below — do **not** jump to force-delete |
| `FailedAttachVolume` (cloud errors: limit, not found, wrong AZ) | Node volume-attach limit reached; volume in a different AZ than the node (zonal PV vs multi-AZ nodes) | `kubectl get volumeattachment`; for zonal: the PV's node affinity names the AZ — the pod must schedule there (this is what `WaitForFirstConsumer` prevents) |
| `FailedMount ... timed out waiting` after successful attach | CSI node plugin down on that node, or device path issues | CSI node DaemonSet pod logs on the node; kubelet log |
| `FailedMount` with fsck/`wrong fs type` | Filesystem corruption or fsType mismatch (raw/foreign filesystem on the volume) | fsck output is in the event; repair is a data-plane operation — confirmation required |
| `FailedMount` permission/`applying fsGroup` slowness | Ownership change on huge volumes, or backend doesn't support fsGroup | `fsGroupChangePolicy: OnRootMismatch` avoids full re-chown |
| NFS: `mount.nfs: Connection timed out` | Server/export unreachable from the node, export options | Test from the node's network context, check export list — not a Kubernetes-layer problem |

### Multi-Attach walkthrough (node failure)

1. `kubectl get volumeattachment | grep <pv>` — which node holds the attachment?
2. What state is that node in? `kubectl get node` — `NotReady` since when? Is the machine
   actually dead, or partitioned/rebooting?
3. If the node is **coming back**: wait — kubelet reconciles, the attachment releases, the new
   pod attaches. Total loss window is minutes.
4. If the node is **confirmed dead** (machine gone): deleting the Node object (or
   force-deleting its pods) releases the attachment. **Mutating and risky** — if the node is
   actually alive and writing, forced reattach elsewhere = two writers = filesystem corruption.
   State this trade-off explicitly; the user decides.
5. RWX would sidestep this, but most block CSI drivers are RWO — "make it RWX" is a design
   change, not a fix.

## Expansion stuck

- Prereq: SC has `allowVolumeExpansion: true`; shrinking is not supported — a smaller request
  is rejected outright.
- Two stages: backend resize (controller) → filesystem resize (node). PVC condition
  `FileSystemResizePending` = stage two waiting; on older stacks it completes only when the pod
  restarts. `status.capacity` unchanged while `spec` is bigger = still in flight; check CSI
  controller logs for stage-one errors (backend full/quota is common here too).
- Never "fix" a stuck expansion by recreating the volume — that converts a wait into data loss.

## StatefulSet specifics

- `volumeClaimTemplates` are immutable — changing size/class there requires the documented
  dance (edit PVCs directly for size; for class: new PVCs via `--cascade=orphan` recreate) —
  plan it, don't improvise it.
- Scale-down keeps PVCs by design (replica N's data waits for its return). "Orphaned" PVCs
  after a permanent scale-down are a cleanup decision with data impact — list, confirm, then
  delete.
- Pod stuck because *its* PVC is stuck: the pod name→PVC name binding is fixed
  (`data-<sts>-<ordinal>`) — debugging the PVC is debugging the pod.

## Backends

**Longhorn** — volume state in the Longhorn UI/CRDs (`kubectl -n longhorn-system get
volumes.longhorn.io`): `Degraded` = replica(s) lost, rebuilding (works, but redundancy is
reduced — don't panic-detach); `Faulted` = all replicas unusable (salvage is a mutating
recovery — confirmation). Common causes: node disk pressure evicting replicas
(over-provisioning percentage, disk reservation), a node down taking replicas with it, or
`numberOfReplicas` > schedulable disks so rebuilds never place.

**Rook-Ceph** — `kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph status`:
- `HEALTH_WARN` with `mon ... down` / no quorum: mons need majority — 2-of-3 down blocks all IO.
- OSD `nearfull` (85%) warns; `full` (95%) flips the cluster **read-only** — every mount
  suddenly erroring on writes with a "healthy-looking" workload is this. Capacity, not bugs.
- `pg ... inactive/incomplete`: data placement broken (too few OSDs for replica count/failure
  domain) — placement math, not restarts, fixes it.

**local-path / hostPath** — data is on one node by definition: the pod must return to that
node (node affinity on the PV); node gone = data gone (that's the contract). PVC Pending with
local-path usually means no pod yet (`WaitForFirstConsumer`) or the node's path/disk is full.
