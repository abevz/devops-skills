# Workload autoscaling: HPA, VPA, KEDA

## HPA (autoscaling/v2)

**The math**: `desiredReplicas = ceil(current × currentMetric / targetMetric)`, evaluated per
sync (15s default). CPU/memory `Utilization` targets are a percentage **of requests** — pods
without requests make the metric `<unknown>` and the HPA inert; requests set to a fraction of
real usage make 70% targets fire constantly.

**Metric types**: `Resource` (CPU/memory from metrics-server), `ContainerResource` (one
container instead of the pod sum — use for sidecar-heavy pods where the sidecar drowns the
signal), `Pods` (custom per-pod), `Object` (one metric for the whole target, e.g. ingress RPS),
`External` (from outside the cluster — usually better served by KEDA). Multiple metrics =
highest desired count wins.

**Behavior tuning** (`spec.behavior`) — the thrash controls:

- `scaleDown.stabilizationWindowSeconds`: default 300 — the HPA uses the *highest* desired
  count of the window; oscillating metrics hold replicas up.
- `scaleUp.stabilizationWindowSeconds`: default 0 — up is immediate by design.
- Policies rate-limit change: default scale-up is max(4 pods, 100%) per 15s; scale-down 100%
  per 15s (bounded by stabilization). A spiky workload usually wants a longer down-window or a
  percent-based down policy, not a pinned replica count.

**Failure modes**:

| Symptom | Cause |
|---|---|
| `<unknown>` target, no scaling, `FailedGetResourceMetric` | metrics-server missing/broken, or no requests on pods |
| Scales on memory but never down | Memory doesn't return to baseline in most runtimes — memory is usually a wrong HPA signal; consider VPA or a custom metric |
| Thrashing up/down | Stabilization/policies at defaults with an oscillating metric; or two controllers own replicas (see KEDA below) |
| Never reaches minReplicas at night | That's correct behavior — HPA can't go below min; scale-to-zero needs KEDA |
| Scaling correct but pods stay Pending | Node layer can't place them — `references/node-autoscaling.md` |

`kubectl describe hpa <name>` events show every decision with the metric reading that drove
it — always the first read.

## VPA

Modes: `Off` (recommendations only — safe default, read them in `kubectl describe vpa`),
`Initial` (applies at pod creation only), `Auto`/`Recreate` (evicts pods to resize — respects
PDBs but *is* disruption). Newer stacks (K8s 1.33+ in-place resize, recent VPA) can resize
without recreation — verify both versions before promising no-restart resizing.

**The HPA conflict**: VPA changing requests changes the denominator of HPA's CPU% at the same
time HPA changes replicas — the loop fights itself. Rule: HPA and VPA never manage the same
resource metric on the same workload (HPA-on-custom-metric + VPA-on-resources is the legal
combination). Start every VPA in `Off` mode; graduate deliberately.

## KEDA

KEDA wraps HPA: a `ScaledObject` names the workload, triggers, and bounds; KEDA creates and
owns the HPA (**never** hand-create an HPA for the same workload — instant replica-count
fighting), and adds the piece HPA can't do: **activation** — 0→1 on the trigger's
`activationThreshold`, then the HPA math handles 1→N. `minReplicaCount: 0` is scale-to-zero;
`cooldownPeriod` (default 300s) sets how long after the last activity the 1→0 step waits.

- 70+ scalers: queue depth (SQS/RabbitMQ/Kafka consumer lag), Prometheus query, cron
  (pre-scale for known peaks), cloud metrics. A queue consumer scaling on lag beats any CPU
  proxy for the same workload.
- `ScaledJob` spawns Jobs per work item instead of scaling a Deployment — batch shape.
- Credentials go through `TriggerAuthentication`/`ClusterTriggerAuthentication` (pod identity,
  secret refs) — never inline in the ScaledObject.
- Failure modes: scaler auth errors leave the workload stuck at current count (check
  `kubectl get scaledobject` READY/ACTIVE columns + keda-operator logs); an aggressive
  activation threshold + short cooldown = 0↔1 flapping with cold-start latency every cycle.

## Choosing on the workload axis

| Situation | Answer |
|---|---|
| CPU-bound request/response service | HPA on CPU (requests verified first) |
| Sidecar-heavy pod, main container is the signal | HPA with `ContainerResource` |
| Queue/stream consumer | KEDA on lag/depth, scale-to-zero if cold start is acceptable |
| Known daily peak | KEDA cron trigger alongside the load trigger |
| Right-sizing chronically mis-requested workloads | VPA `Off` → read → apply; never with HPA on the same metric |
| Batch/one-Job-per-item | KEDA `ScaledJob` |
