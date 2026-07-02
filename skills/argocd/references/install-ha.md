# Argo CD install, HA, and scaling

Self-authored reference (no third-party source). Components, HA topology, controller sharding,
repo-server sizing, upgrades. Deployment-planning: propose Helm values, never apply.

## Components (what scales how)

| Component | Role | State | Scale |
|---|---|---|---|
| `argocd-server` | API + UI | Stateless | replicas (behind a Service/ingress) |
| `argocd-repo-server` | Renders manifests (helm/kustomize/plugins) | Stateless, CPU+mem heavy | replicas — the usual bottleneck |
| `argocd-application-controller` | Reconciles apps → clusters | StatefulSet | shard by cluster (replicas + sharding) |
| `argocd-redis` | Cache (rendered manifests, cluster state) | Ephemeral cache | HA via redis-ha / sentinel |
| `argocd-applicationset-controller` | Generates Applications | Stateless | usually 1 (leader-elected) |
| `argocd-notifications-controller` | Sends notifications | Stateless | 1 |
| `argocd-dex-server` | SSO connector (if using Dex) | Stateless | 1 |

Redis is a **cache** — losing it isn't data loss (rebuilt), but losing it under load causes a
reconcile storm; HA redis avoids the blip.

## HA install

Use the `argo-cd` Helm chart with HA values (or the `ha/` manifests). Key values:

```yaml
redis-ha: {enabled: true}                      # replaces single redis; needs 3 nodes for quorum
controller:
  replicas: 2                                   # requires sharding (below)
server:
  replicas: 2
  autoscaling: {enabled: true, minReplicas: 2}
repoServer:
  replicas: 3                                   # scale to render throughput
  autoscaling: {enabled: true}
applicationSet: {replicas: 1}
```

- `redis-ha` needs 3 schedulable nodes (anti-affinity) — on a 2-node cluster it won't achieve
  quorum; note that in capacity planning.
- Don't run HA on a tiny setup for its own sake — a single-replica Argo CD managing a few apps is
  legitimate; HA earns its complexity at scale/criticality.

## Controller sharding (the part people get wrong)

The application-controller shards **by cluster**, not by app. `replicas > 1` does nothing unless
sharding is enabled and there are multiple clusters to spread:

```yaml
controller:
  replicas: 3
  env:
    - {name: ARGOCD_CONTROLLER_REPLICAS, value: "3"}   # must match replicas
  # sharding algorithm: legacy | round-robin | consistent-hashing (newer, rebalances less)
```

- `ARGOCD_CONTROLLER_REPLICAS` must equal the replica count or shards misassign.
- Consistent-hashing (newer versions) rebalances fewer clusters when a replica changes — prefer
  it over legacy round-robin for many clusters.
- Single-cluster Argo CD gets **no benefit** from controller replicas — one shard owns the one
  cluster. Scale repo-server instead if reconcile is slow (usually it's rendering, not the
  controller).

## repo-server sizing — the real bottleneck

Rendering (helm template / kustomize build) happens here, per app, and it's CPU+memory hungry:

- Scale `repoServer.replicas` for many apps / frequent syncs; each concurrent render holds memory
  proportional to the chart/overlay size.
- `--parallelismLimit` caps concurrent renders per repo-server (prevents OOM under a thundering
  herd of syncs).
- `ARGOCD_EXEC_TIMEOUT` (default 90s) — big charts/plugins hit it; raise deliberately.
- Mount enough memory; repo-server OOM shows as flapping `ComparisonError`/render failures
  (cross-ref `argocd-debug`).

## Bootstrap and self-management

- Common pattern: install Argo CD, then have Argo CD manage itself (an `Application` pointing at
  the Argo CD Helm chart / manifests in git) — "app-of-apps" including the platform. Careful: a
  bad self-managed change can break the thing applying the fix; keep a break-glass manifest apply
  path documented.
- CRDs (`Application`, `ApplicationSet`, `AppProject`) install with the chart; on upgrade, CRDs
  are a deliberate step (chart doesn't always upgrade them in place).

## Upgrades

- Read the release notes every minor — Argo CD occasionally changes RBAC defaults, repo-server
  behavior, or resource-tracking. Pin the chart version.
- Upgrade path is roughly: CRDs → components; one minor at a time for big jumps.
- Test in a non-prod Argo CD first; a broken repo-server upgrade stops all syncs cluster-wide.

## Verify (read-only)

```bash
argocd version
kubectl get pods -n argocd
argocd admin settings validate           # config sanity
kubectl get statefulset argocd-application-controller -n argocd -o jsonpath='{.spec.replicas}'
```
