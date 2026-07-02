# Cilium Cluster Mesh

Self-authored reference (no third-party source). Connecting multiple clusters into one
service/identity/policy fabric. Deployment-planning: propose config, don't run
`clustermesh enable`.

## What it does

Cluster Mesh links 2+ Cilium clusters so they share identities and can route/load-balance
services across cluster boundaries: cross-cluster service discovery, HA failover, and network
policy that spans clusters.

## Hard prerequisites (check these first — they cause most failures)

| Requirement | Why |
|---|---|
| Unique `cluster.name` **and** `cluster.id` (1–255) per cluster | Identity allocation collides otherwise |
| Non-overlapping Pod CIDRs across clusters | Routing ambiguity (newer versions relax this, but plan for it) |
| Node-to-node reachability between clusters (pod/node IPs routable, or via the clustermesh-apiserver LoadBalancer) | The data path must actually connect |
| Shared/compatible CA (or explicit trust) between clusters | mTLS between clustermesh-apiservers |
| Same datapath assumptions (encryption on all or none) | Mixed = broken/insecure cross-cluster traffic |

Overlapping PodCIDR is the #1 blocker; unique cluster IDs the #2. Verify both before any config.

## Shape

Each cluster runs a `clustermesh-apiserver` (exposed via LoadBalancer/NodePort); clusters are
paired by exchanging its access secret. Config lives in Helm values (`clustermesh.*`) — GitOps
it; the `cilium clustermesh enable` / `connect` CLI is for labs.

## Global services

```yaml
# Service becomes cross-cluster load-balanced by annotation:
metadata:
  annotations:
    service.cilium.io/global: "true"
    service.cilium.io/affinity: "local"   # prefer local-cluster backends, spill over remote
```

- A global Service with the same name/namespace in both clusters gets its endpoints merged;
  traffic load-balances across clusters (with `affinity: local` for locality, failover to remote).
- `service.cilium.io/shared: "false"` on one side to consume-but-not-export.
- Use cases: active/active across regions, failover, shared platform services. Not a substitute
  for a real global load balancer at the edge — this is east-west, in-cluster-originated traffic.

## Operational reality — recommend deliberately

- Cluster Mesh is real added complexity and blast radius: an identity/CA/CIDR mistake can affect
  multiple clusters at once. Recommend it only for genuine multi-cluster service needs (HA,
  locality), not "we have two clusters so let's mesh them."
- Cross-cluster policy: CNP can select `remote-node`/cluster entities and specific clusters —
  powerful, and easy to write a rule that behaves differently per cluster; validate with Hubble
  on both sides.
- Network policy default-deny + Cluster Mesh: remember cross-cluster traffic must be explicitly
  allowed on both ends, same as intra-cluster egress/ingress pairing.

## Verify (read-only)

```bash
cilium clustermesh status          # connected clusters, tunnel state, endpoint counts
cilium status | grep -i clustermesh
# per-service: check merged endpoints exist for the global service
```

Rollout: enable on a non-prod pair first, validate global-service failover with Hubble, then
extend — never mesh production clusters as the first exercise.
