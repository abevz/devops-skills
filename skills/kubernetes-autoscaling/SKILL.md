---
name: kubernetes-autoscaling
description: Use when designing or reviewing Kubernetes autoscaling — HPA behavior/metrics, VPA, KEDA event-driven scaling and scale-to-zero, cluster-autoscaler, and Karpenter node provisioning. Mention "autoscaling", "hpa", "vpa", "keda", "scaledobject", "karpenter", "nodepool", "cluster-autoscaler", "scale to zero" as triggers.
license: MIT
compatibility: Works with Claude Code, Codex-style agents, CodeWhale, OpenCode, and other agents that support Agent Skills-style instructions.
---

# Kubernetes Autoscaling

## TL;DR checklist

- [ ] Identify workload scaling versus node provisioning.
- [ ] Check resource requests and the metric source first.
- [ ] Match scaling policy and disruption protection to the bottleneck.

## Answer must include

Before proposing a metric, threshold, or replica floor, establish the measured bottleneck,
honest resource requests, available metrics, and node capacity and disruption constraints.
If these are unknown, give read-only checks and decision branches instead of numeric manifests.

## Key read-only checks

- Read requests, HPA/KEDA status, Pending events, and node provisioner decisions.

## Common pitfalls

- Do not tune CPU scaling when the queue or another external signal is the bottleneck.

## Agent procedure

Follow the [Workflow](#workflow), [Safety rules](#safety-rules), and [Quality checklist](#quality-checklist) below.

## Quick references

- [references/workload-autoscaling.md](references/workload-autoscaling.md)
- [references/node-autoscaling.md](references/node-autoscaling.md)

Last verified: unverified

## When to use

Use when adding, tuning, or reviewing autoscaling at either layer: **workload** (HPA, VPA, KEDA
ScaledObject/ScaledJob) or **node** (cluster-autoscaler, Karpenter). Also use when scaling
misbehaves by design — thrashing replicas, pods Pending with "0/N nodes available", scale-to-zero
questions. For a workload that's failing rather than mis-scaling, start with `kubernetes-debug`;
for whether the service can survive scaling events at all (PDBs, graceful shutdown), see
`production-readiness`.

## Goal

Autoscaling that responds to the actual bottleneck signal, doesn't fight itself (HPA vs VPA,
KEDA vs hand-written HPA), respects disruption budgets, and — at the node layer — provisions
right-sized capacity instead of whatever the fixed node group happens to offer.

## Workflow

1. **Scope to a layer** and load its reference:

   | Layer | Reference |
   |---|---|
   | Workload: HPA (metrics, behavior), VPA (modes, conflicts), KEDA (scalers, scale-to-zero) | `references/workload-autoscaling.md` |
   | Node: cluster-autoscaler vs Karpenter, consolidation, disruption control | `references/node-autoscaling.md` |

2. **Confirm the prerequisites before any recommendation** — autoscaling is built on two
   foundations that are broken more often than the autoscaler itself:
   - **Resource requests set and honest** — HPA CPU% is a percentage *of requests*; Karpenter
     and cluster-autoscaler size capacity purely from requests. Wrong requests = wrong scaling
     everywhere, and no tuning fixes it.
   - **The metric exists** — metrics-server for resource metrics; a Prometheus adapter or KEDA
     for anything custom/external. `kubectl top pods` failing means HPA is blind.
3. **Scale on the bottleneck signal, not CPU by habit** — a queue consumer scales on queue
   depth (KEDA), a latency-sensitive API might scale on RPS/latency (custom metric), CPU only
   when CPU is genuinely the saturating resource.
4. **One owner per axis** — never two controllers on the same axis: KEDA *creates* an HPA
   (never both on one workload); VPA in recreate mode must not manage the same resource an HPA
   scales on. Horizontal for load, vertical for right-sizing — pick deliberately per workload.
5. **Plan for the disruption scaling causes** — scale-down and node consolidation evict pods:
   PDBs, graceful shutdown, and `do-not-disrupt` annotations for the genuinely
   interruption-intolerant are part of the autoscaling design, not an afterthought.
6. **Homelab/bare-metal reality check** — node autoscaling presumes an elastic provider; on
   fixed hardware the node layer is capacity planning, and only the workload layer applies.
7. **GitOps the config** — HPA/VPA/ScaledObject/NodePool are declarative and belong in git.
8. **Verify read-only** — `kubectl describe hpa` (events show every decision and its metric
   reading), `kubectl get scaledobject`, KEDA operator logs, `kubectl get nodeclaims` /
   cluster-autoscaler status ConfigMap, and Pending-pod events for the node layer.

## References

- `references/workload-autoscaling.md` — HPA metrics/behavior math and failure modes, VPA
  modes and the HPA conflict, KEDA scalers/activation/scale-to-zero.
- `references/node-autoscaling.md` — cluster-autoscaler vs Karpenter decision table,
  NodePool/consolidation/disruption design, protection annotations.

## Safety rules

- Design/review skill: do not apply HPA/VPA/ScaledObject/NodePool changes or delete/cordon
  nodes on a live cluster — propose manifests for review.
- Scale-down settings are eviction settings — aggressive consolidation or a missing PDB turns
  "optimize cost" into an availability incident; frame scale-down changes with their disruption
  blast radius.
- Never propose disabling autoscaling (or pinning replicas) as a quick fix for thrashing —
  tune stabilization/policies against the observed oscillation instead.
- Scale-to-zero means cold starts and (for queues) latency until activation — call the
  trade-off out explicitly wherever it's proposed.
- Cloud credentials for KEDA scalers and Karpenter are secrets — reference a secrets mechanism
  (`secrets-management`), never inline.

## Output format

```
## Layer & baseline (requests honest? metric source? current scalers? node groups/provider)
## Recommendation (what scales on what signal, and explicitly what NOT to add)
## Prerequisites & blast radius (metrics pipeline, PDBs, disruption settings)
## Proposed config (HPA/ScaledObject/NodePool manifests — for review)
## Rollout & rollback
## Verification (read-only: describe hpa events, nodeclaims, Pending-pod events)
```

## Quality checklist

- [ ] Resource requests were verified honest before any scaling math
- [ ] The scaling signal matches the actual bottleneck, not CPU-by-default
- [ ] No axis has two owners (KEDA vs HPA, VPA-recreate vs HPA on the same metric)
- [ ] Disruption is designed: PDBs / graceful shutdown / do-not-disrupt where warranted
- [ ] Node-layer advice matches the environment (elastic cloud vs fixed hardware)
- [ ] Nothing was applied to a live cluster; verification is read-only
