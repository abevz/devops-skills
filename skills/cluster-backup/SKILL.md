---
name: cluster-backup
description: Use when designing or reviewing Kubernetes cluster backup and disaster recovery — what GitOps does and doesn't rebuild, Velero (schedules, CSI snapshots vs file-system backup, restore design), etcd snapshots for self-hosted clusters, and restore testing. Mention "velero", "cluster backup", "disaster recovery", "etcd snapshot", "restore", "backup strategy" as triggers.
license: MIT
compatibility: Works with Claude Code, Codex-style agents, CodeWhale, OpenCode, and other agents that support Agent Skills-style instructions.
---

# Cluster Backup & DR

## When to use

Use when designing, reviewing, or questioning a cluster's backup/DR posture: "do we even need
backups if we have GitOps?", Velero setup and schedules, persistent-volume data protection,
etcd snapshots on self-hosted clusters, and restore/DR planning. For one component's own DR
story, the umbrellas own it (`argocd`, `vault`, `observability-stack`); for writing the actual
DR runbook, hand off to `runbook-writer`. This skill owns the cluster-wide inventory and the
Velero/etcd layer.

## Goal

A backup posture where every stateful thing in the cluster is either provably rebuildable from
git or covered by a scheduled, monitored, **restore-tested** backup — with RPO/RTO stated,
storage off-cluster, and no category silently falling through the GitOps gap.

## Workflow

1. **Inventory what git does NOT rebuild** — "everything is in GitOps" covers desired state
   only. Walk the gap list explicitly:
   - **PV data** — databases, queues, uploads; git has the PVC, not the bytes.
   - **Secrets not in git** — anything created by hand, by operators, or by controllers
     (repo/cluster creds, webhook CAs, sealed-secrets private key, Vault storage).
   - **In-cluster state with no source of truth** — CRD objects written by apps/operators at
     runtime, cert-manager ACME account keys, controller-issued identities.
   - **etcd/control plane** (self-hosted only) — certificates, the API state itself.
   Each item lands in a bucket: *git rebuilds it* / *backup covers it* / *uncovered (finding)*.
2. **State RPO/RTO per bucket** — "how much can we lose / how long can restore take" decides
   schedule frequency, snapshot vs file-copy, and whether cross-region copies are warranted.
   A homelab tolerating 24h loss and a prod database tolerating 15min are different designs.
3. **Design the Velero layer** — load `references/velero.md`: backup storage location
   off-cluster, schedules with TTLs, CSI snapshots vs file-system backup chosen per volume
   type, hooks for app-consistent database backups, and restore semantics (namespace mapping,
   existing-resource policy).
4. **Self-hosted control plane** — etcd snapshots on a schedule + the PKI directory; a managed
   control plane (EKS/GKE/AKS) removes this layer but none of the others.
5. **A backup is a rumor until restored** — the design must include a restore test with a
   cadence (restore into a scratch namespace/cluster, verify the application actually comes up
   and the data reads). An untested backup pipeline counts as *uncovered* in the inventory.
6. **Monitor the pipeline** — backup failures alert (`velero backup get` states,
   PartiallyFailed drift, last-success age metric); a silently failing schedule is the worst
   outcome in this domain.
7. **Hand off the runbook** — the restore procedure, owners, and decision points go to
   `runbook-writer`; the incident that triggers it goes to `incident-analysis` afterwards.

## References

`references/velero.md` — install/BSL/schedule design, CSI snapshot vs node-agent file-system
backup decision table, hooks for consistency, restore semantics, failure modes, monitoring,
and the etcd snapshot section for self-hosted clusters.

## Safety rules

- Design/review skill: do not run `velero backup create/delete`, `velero restore create`, or
  `etcdctl snapshot restore` against a live cluster — propose schedules/manifests and the
  restore *plan*; the user executes.
- **Restores overwrite state** — a restore into an occupied namespace/cluster can clobber
  newer data with older data; every restore proposal states its collision policy and what it
  will overwrite, and requires explicit confirmation.
- `etcdctl snapshot restore` rebuilds the control plane's view of everything — treat as the
  most destructive operation in Kubernetes; plan-only, never suggested casually.
- Backup deletion (or TTL shortening) destroys recovery points — flag retention reductions as
  destructive changes.
- Object-storage credentials for backup locations are secrets — reference a secrets mechanism
  (`secrets-management`); backups themselves contain Secrets in plaintext by default — the
  bucket's access policy and encryption are part of the review, not an afterthought.

## Output format

```
## Inventory (git-rebuildable / backup-covered / UNCOVERED, per stateful thing)
## RPO/RTO (stated per bucket, driving the design)
## Proposed design (BSL, schedules, snapshot-vs-FSB per volume, hooks, etcd layer)
## Restore plan & test cadence (how, into what, verified by what)
## Monitoring (what alerts when backups stop working)
## Open risks (what remains uncovered and why)
```

## Quality checklist

- [ ] The GitOps gap inventory was walked explicitly; every stateful item has a bucket
- [ ] RPO/RTO stated and the schedule/method actually meets them
- [ ] Snapshot vs file-system backup chosen per volume type with the trade-off stated
- [ ] Database-class workloads have consistency hooks (or a stated reason they don't need them)
- [ ] A restore test with a cadence is part of the design, not a suggestion
- [ ] Backup storage is off-cluster, access-controlled, and its Secrets exposure addressed
- [ ] Nothing was backed up/restored/deleted by the agent; all operations are proposed
