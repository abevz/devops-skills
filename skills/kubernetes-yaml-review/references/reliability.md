# Reliability deep-dive: probes, rollouts, resources

Distilled from LukasNiessen/kubernetes-skill (`references/fragile-rollouts.md`,
`references/resource-starvation.md`) — the two highest-value sources found during third-party
review. Use this when reviewing probes, rollout strategy, or resource blocks in depth.

## Probe semantics — the rule that prevents cascading outages

- **Liveness** = "is the process alive / not deadlocked?" Failure → kubelet kills the container.
  **NEVER check external dependencies here.** The cascade when you do: DB goes down → liveness
  fails on *all* pods simultaneously → kubelet restarts all of them → DB still down → restart
  loop across the whole service until the DB recovers. A liveness probe checking only "is the
  event loop responsive" would have kept every pod alive through the outage.
- **Readiness** = "can this pod serve traffic right now?" Failure → removed from endpoints, NOT
  killed. This IS where dependency checks belong: DB down → stop receiving requests → recover
  when it returns.
- **Startup** = "has initialization finished?" While it runs, liveness/readiness are disabled.
  Required for anything with >10s startup (JVM warmup, model loading). Budget:
  `failureThreshold × periodSeconds ≥ max startup time` (e.g. 30 × 5s = 150s budget).

Timing guardrails: liveness `failureThreshold ≥ 3` (never 1 — one blip kills the pod),
`timeoutSeconds < periodSeconds`, readiness faster than liveness (it controls traffic).

## Graceful shutdown — why pods drop requests on deploy

Termination steps run **in parallel**: endpoint removal is async with SIGTERM, so the pod can
still receive traffic for a few seconds *after* SIGTERM arrives. Review for:

- `lifecycle.preStop: sleep 3-5s` — buys time for endpoint de-registration before shutdown
  begins. Its absence is the usual cause of 502s during rollouts.
- `terminationGracePeriodSeconds` > the app's real drain time (default 30s; long-polling/
  streaming workloads need more).
- Exec-form entrypoint so SIGTERM actually reaches the app (PID 1 shell swallowing signals).

## Rolling update strategy

- Zero-downtime requires `maxUnavailable: 0` + `maxSurge ≥ 1`. Watch the arithmetic:
  2 replicas with `maxUnavailable: 1` = 50% capacity loss during every deploy.
- `minReadySeconds: 10+` catches crash-after-start pods before the rollout proceeds — cheap and
  almost always missing.
- `revisionHistoryLimit` set (default 10 is fine; unbounded old ReplicaSets are clutter).
- Init containers (not liveness probes, not retry loops in-app) are the right place to wait for
  dependencies at startup.

## Resources — QoS and the CPU-limit question

| QoS | Condition | Evicted |
|---|---|---|
| Guaranteed | requests == limits (cpu+mem) on every container | last |
| Burstable | any request ≠ limit | middle |
| BestEffort | nothing set | **first — never acceptable in production** |

- **Memory: always set a limit** (incompressible; overflow = OOMKill). Headroom 25–50% above
  observed p99, not requests == limits by reflex.
- **CPU: limits are a deliberate choice, not a default.** CPU limits cause CFS throttling —
  invisible latency spikes that don't show up as errors. Prefer request-only CPU for latency-
  sensitive services; set CPU limits only for multi-tenant fairness, batch workloads, or when
  Guaranteed QoS is required. Flag cargo-culted `cpu: "1"` limits in review.
- Requests must come from observed usage (`kubectl top pods --containers`), not round numbers.

## Cross-resource consistency checks

These bite because each object looks fine alone:

- **HPA `minReplicas` ≥ PDB `minAvailable`** — otherwise scale-down violates the disruption
  budget and blocks node drains.
- **PDB `minAvailable` < `replicas`** — equal values block ALL voluntary disruption, including
  cluster upgrades.
- **HPA target 60–80% utilization** — 90% leaves no headroom for the scale-up lag; 30% is
  paying for idle.
- Replicas > 1 without topologySpreadConstraints/anti-affinity = false redundancy (all replicas
  can land on one node).
- LimitRange in the namespace as the safety net for workloads that slip through review without
  resource blocks.

## Read-only verification one-liners

```bash
# QoS class per pod
kubectl get pods -n <ns> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.qosClass}{"\n"}{end}'

# BestEffort candidates (no requests anywhere)
kubectl get pods -A -o json | jq -r '.items[] | select(.spec.containers[].resources.requests == null) | "\(.metadata.namespace)/\(.metadata.name)"'

# Recent OOMKills
kubectl get pods -A -o json | jq -r '.items[].status.containerStatuses[]? | select(.lastState.terminated.reason == "OOMKilled") | .name'

# Deployments on :latest or untagged images
kubectl get deployments -A -o json | jq -r '.items[] | .metadata.namespace + "/" + .metadata.name as $d | .spec.template.spec.containers[] | select(.image | endswith(":latest") or (contains(":") | not)) | $d + " -> " + .image'

# Containers without readiness probes
kubectl get pods -A -o json | jq -r '.items[] | .metadata.namespace + "/" + .metadata.name as $pod | .spec.containers[] | select(.readinessProbe == null) | $pod + " :" + .name'

# PDBs that currently allow zero disruptions (block drains)
kubectl get pdb -A -o jsonpath='{range .items[*]}{.metadata.namespace}/{.metadata.name}{": allowed="}{.status.disruptionsAllowed}{"\n"}{end}'

# Probe failure events
kubectl get events -n <ns> --field-selector reason=Unhealthy --sort-by=.lastTimestamp
```
