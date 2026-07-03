---
name: storage-debug
description: Use when debugging Kubernetes storage issues — PVC stuck Pending, volume attach/mount failures, Multi-Attach errors, volume expansion stuck, StatefulSet storage problems, or backend issues in Longhorn, Rook-Ceph, NFS, or local-path setups. Mention "pvc pending", "failedattachvolume", "failedmount", "multi-attach error", "volume expansion", "longhorn", "rook", "ceph" as triggers.
license: MIT
compatibility: Works with Claude Code, Codex-style agents, CodeWhale, OpenCode, and other agents that support Agent Skills-style instructions.
---

# Storage Debug

## When to use

Use when persistent storage misbehaves in a Kubernetes cluster: a PVC stuck `Pending`, pods
stuck `ContainerCreating` on volume attach/mount, `Multi-Attach` errors after a node failure,
volume expansion that never completes, StatefulSet storage quirks, or backend-level trouble in
Longhorn, Rook-Ceph, NFS, or local-path storage. For the workload failing for non-storage
reasons, start with `kubernetes-debug`; for designing backup of the data, see `cluster-backup`.

## Goal

Pinpoint which layer is failing — claim/class, provisioner, attach, mount, filesystem, or the
storage backend itself — backed by the event/log evidence of that layer, and propose a fix
without deleting claims, volumes, or data.

## Workflow

1. **Read the events first** — `kubectl describe pvc <name>` and `kubectl describe pod <name>`:
   the event text (`ProvisioningFailed`, `FailedAttachVolume`, `FailedMount`, `Multi-Attach
   error`) names the failing layer directly. Route into `references/playbooks.md` by that text.
2. **Locate the layer** — the storage path is a chain; find the deepest healthy link:
   - **Claim/class**: does the PVC's StorageClass exist? Is there a default at all
     (`kubectl get sc`)? `WaitForFirstConsumer` means Pending-until-pod-schedules is *normal*.
   - **Provisioner**: CSI controller pods running? Their logs hold the create-volume error
     (quota, credentials, backend full).
   - **Attach**: `kubectl get volumeattachment` — is the volume attached to the right node?
   - **Mount**: CSI node plugin (DaemonSet) logs on that node; kubelet events for fsck/fsType/
     permission errors.
   - **Backend**: Longhorn volume/replica state, `ceph status`, NFS server reachability.
3. **Node-failure special case** — `Multi-Attach error` for an RWO volume after a node died is
   the expected protection, not the bug: the old attachment must be released (node recovers,
   or its Node object/pods are deleted) before the new node can attach. Understand which state
   the old node is in before touching anything.
4. **Expansion path** — expansion has two stages (backend resize, then filesystem resize);
   `FileSystemResizePending` on the PVC means stage two is waiting — often for a pod restart on
   older stacks. Never treat a stuck expansion as "recreate the volume."
5. **StatefulSet specifics** — one PVC per replica from `volumeClaimTemplates` (immutable);
   scaling down does not delete PVCs (a returning replica reattaches its old data); a "reset"
   that deletes PVCs is a data-loss decision, never a debug step.
6. **Backend deep-dive** — for Longhorn/Rook-Ceph symptoms, use the backend sections in the
   playbooks: replica health and disk pressure for Longhorn; mon quorum, OSD state, and
   full-ratios for Ceph.
7. **Correlate and conclude** — tie the event text to the specific layer and cause; propose the
   minimal fix and how to verify the volume is healthy end-to-end afterwards.

## References

`references/playbooks.md` — event-text → cause → fix tables per layer (provisioning, attach,
mount, expansion), the node-failure/Multi-Attach walkthrough, and backend sections for
Longhorn, Rook-Ceph, NFS, and local-path. Route there as soon as an event names the failure.

## Safety rules

- Read-only first, always: `kubectl get/describe pvc,pv,sc,volumeattachment`, CSI pod logs,
  `longhornctl`/Longhorn UI state, `ceph status` (via toolbox). None of these mutate state.
- **Never delete a PVC, PV, or VolumeSnapshot as a diagnostic** — depending on `reclaimPolicy`
  that is irreversible data deletion. Any delete is proposed explicitly with its data impact
  stated, and the user confirms.
- Force-detach measures (deleting a Node object, force-deleting pods, removing finalizers)
  break the Multi-Attach protection — only propose them when the old node is *confirmed* dead,
  and say what happens if it isn't (split-brain writes, filesystem corruption).
- Never edit `reclaimPolicy`, filesystem, or backend replica counts as a quick fix; changes to
  data-bearing settings are framed as reviewed changes with rollback.
- Backend repair commands (Longhorn salvage, `ceph osd` mutations) require explicit
  confirmation — they act on the data plane itself.

## Output format

```
## Observations
<claim/class/provisioner/attach/mount/backend state, from which command>

## Evidence
<the event text / log line that names the failing layer, quoted>

## Root cause
## Fix
<manifest or action, with data impact stated, marked as requiring user confirmation>

## Verification
<end-to-end check: pod mounts, writes succeed, backend reports healthy>
```

## Quality checklist

- [ ] The failing layer was located by event/log evidence, not guessed
- [ ] `WaitForFirstConsumer` semantics were checked before calling a Pending PVC broken
- [ ] For Multi-Attach/node-failure: the old node's true state was established before any
      force-detach was proposed
- [ ] No PVC/PV/snapshot deletion was proposed as a diagnostic; every mutating step states its
      data impact and requires confirmation
- [ ] The fix includes an end-to-end verification (mount + write + backend health)
