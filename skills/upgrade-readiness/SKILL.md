---
name: upgrade-readiness
description: Use when planning or reviewing a Kubernetes cluster version upgrade — version skew rules, deprecated/removed API detection (pluto/kubent), addon compatibility (CNI/CSI/ingress/operators), PDB/drain readiness, and staged per-minor upgrade plans for kubeadm/k3s/Talos or managed EKS/GKE/AKS. Mention "cluster upgrade", "kubernetes upgrade", "api deprecation", "kubent", "pluto", "version skew", "upgrade 1.x to 1.y" as triggers.
license: MIT
compatibility: Works with Claude Code, Codex-style agents, CodeWhale, OpenCode, and other agents that support Agent Skills-style instructions.
---

# Upgrade Readiness

## When to use

Use before a Kubernetes cluster upgrade (or when someone proposes one): assess whether the
cluster, its addons, and its workloads will survive the hop, and produce a staged plan. This is
the review/assessment view — the actual `kubeadm upgrade`/node drains/managed-pool rollouts are
executed by the user, never by the agent. For the generic phased-change methodology see
`migration-plan`; for the backup precondition see `cluster-backup`; for writing the operator's
runbook see `runbook-writer`.

## Goal

An upgrade that is boring: every removed API found in git before the control plane finds it in
production, every addon inside its supported range for the target version, drains that
complete instead of deadlocking, one minor per hop, and a stated (honest) rollback story.

## Workflow

1. **Baseline** — current and target versions (control plane, per-node-pool kubelets, addons),
   distro/provider (kubeadm, k3s, Talos, EKS/GKE/AKS), and what drives the deadline (EOL,
   compliance, a feature). Multiple minors between current and target = multiple sequential
   hops in the plan, never one jump.
2. **Skew math first** — the rules that shape the order (details and the full table in
   `references/upgrade-checks.md`): control plane goes first, one minor at a time; kubelets
   may trail the API server by up to three minors (n-3), so node pools can lag but never lead;
   upgrading kube-proxy with its node's kubelet is an operational preference, not the upstream
   skew limit. Anything violating skew *today* is a finding before any
   upgrade starts.
3. **Removed-API scan** — run `pluto`/`kubent` (read-only) against live resources AND against
   the GitOps repo/Helm releases for the *target* version. The fix always lands in git first
   (manifests, chart versions), then reconciles — never hand-patched live. Removed APIs found
   only in released Helm metadata still block `helm upgrade` later; scan those too.
4. **Addon compatibility matrix** — CNI, CSI drivers, ingress/gateway controller,
   cert-manager, observability agents, and every operator each publish a supported-Kubernetes
   range. Build the explicit table: addon → current version → supported range → action
   (upgrade before / after / none). The CNI row is load-bearing (see the `cilium`/`istio`
   umbrellas for their own upgrade mechanics).
5. **Drain readiness** — node upgrades evict everything: PDB coverage for what must stay up,
   but also the inverse check — a PDB with `maxUnavailable: 0` (or minAvailable == replicas)
   deadlocks the drain; singletons, local-storage pods, and long jobs need a stated plan
   (move, tolerate the restart, or schedule around them). Surge capacity for the temporary
   node-count bump must exist (quota, hardware).
6. **Backup precondition** — an etcd snapshot / Velero backup taken and verified immediately
   before each hop is a gate, not a suggestion — because the rollback story is: **control
   planes don't downgrade**; going back means restoring state (`cluster-backup`). Say this
   plainly in the plan.
7. **Stage the plan per hop** — for each minor: control plane → addons that must follow →
   node pools (canary pool first, then the rest, respecting PDBs) → conformance/smoke checks →
   only then the next hop. Managed platforms reshape the mechanics (release channels,
   maintenance windows, node-image vs version upgrades) — per-provider notes in the reference.
8. **Verify read-only** — `kubectl version`, `kubectl get nodes` (version column),
   `pluto detect-all-in-cluster`, `kubent`, addon version listings, `kubectl get pdb -A`.
   Nothing in this skill's workflow mutates the cluster.

## References

`references/upgrade-checks.md` — the skew table, pluto/kubent invocations (cluster, files,
Helm), the addon-matrix template, PDB drain-deadlock checks, canary-pool pattern, and
per-distro/provider notes (kubeadm, k3s, Talos, EKS/GKE/AKS).

## Safety rules

- Assessment/planning skill: do not run `kubeadm upgrade`, drain/cordon nodes, upgrade
  addons, or trigger managed-pool rollouts — produce the readiness report and plan; the user
  executes.
- Never plan a multi-minor jump for the control plane; sequential hops only, each with its own
  backup gate and verification.
- A missing/failed backup before a hop stops the plan — state it as a hard gate, not advice.
- Rollback honesty: never present "roll back the control plane" as an option; the real options
  are restore-from-backup or fix-forward, each with its cost stated.
- API-removal fixes go to git (manifests/charts), never `kubectl edit` on live objects that
  GitOps will fight.

## Output format

```
## Baseline (current/target per component, distro, deadline driver)
## Findings (skew violations, removed APIs with locations in git, addon range breaks, drain risks)
## Addon matrix (addon → version → supported range → action & order)
## Staged plan (per minor hop: backup gate → control plane → addons → canary pool → pools → checks)
## Rollback reality (restore path per hop, what fix-forward looks like)
## Verification (read-only checks per stage)
```

## Quality checklist

- [ ] One minor per control-plane hop; skew rules stated and checked against today's state
- [ ] pluto/kubent run against cluster AND git/Helm state for the target version; fixes routed to git
- [ ] Addon matrix is explicit (every CNI/CSI/ingress/operator row filled), with upgrade order
- [ ] Drain readiness checked both ways (coverage AND deadlock); singletons/local-storage addressed
- [ ] Backup-before-hop is a gate, and the rollback story is restore/fix-forward, honestly stated
- [ ] Nothing was upgraded, drained, or edited live by the agent
