# Istio traffic management

Self-authored reference (no third-party source). Routing, canary, resilience. (Debugging "route
not taking effect" / 503s → `istio-debug`.) Propose config; validate with `istioctl analyze`.

## The core resources

| Resource | Role |
|---|---|
| `VirtualService` | Routing rules: match → route (weights), retries, timeout, fault, mirror |
| `DestinationRule` | Per-destination policy: subsets, load balancing, connection pool, outlier detection, TLS mode |
| `Gateway` | Edge L4-6 listener (ingress/egress) — pairs with a VirtualService |
| `ServiceEntry` | Register an external service into the mesh (so policy/telemetry apply) |
| `Sidecar` | Scope a workload's egress/visibility (perf: limit config pushed to sidecars) |

VirtualService + DestinationRule are a pair: the VS routes to a **subset**, the DR **defines**
that subset (by pod labels). A VS routing to an undefined subset = no endpoints / 503 `UH`
(cross-ref `istio-debug`). This is the most common traffic-config mistake.

## Canary / traffic splitting

```yaml
# DestinationRule defines subsets; VirtualService weights them
kind: DestinationRule
spec: {host: reviews, subsets: [{name: v1, labels: {version: v1}}, {name: v2, labels: {version: v2}}]}
---
kind: VirtualService
spec:
  hosts: [reviews]
  http:
    - route:
        - {destination: {host: reviews, subset: v1}, weight: 90}
        - {destination: {host: reviews, subset: v2}, weight: 10}
```

- Weighted subsets for canary; header/`match`-based routing for targeted testing (route my
  test header to v2). Progressive delivery tools (Argo Rollouts/Flagger) automate shifting these
  weights against metrics — recommend that over hand-editing weights for real canaries.

## Resilience (DestinationRule trafficPolicy)

```yaml
trafficPolicy:
  connectionPool:
    tcp: {maxConnections: 100}
    http: {http2MaxRequests: 1000, maxRequestsPerConnection: 10}
  outlierDetection:                      # = circuit breaking / ejection
    consecutive5xxErrors: 5
    interval: 30s
    baseEjectionTime: 30s
    maxEjectionPercent: 50
```

- `outlierDetection` ejects failing endpoints (circuit breaking). Set `maxEjectionPercent` sanely
  — 100% can eject every backend under a broad fault and self-inflict an outage (shows as `UO`
  in `istio-debug`).
- `connectionPool` limits protect upstreams; too tight self-inflicts 503s under load. Size from
  real concurrency.
- Retries/timeouts on the VirtualService:
  ```yaml
  http: [{ retries: {attempts: 3, perTryTimeout: 2s, retryOn: "5xx,reset,connect-failure"},
          timeout: 10s }]
  ```
  Retry budgets multiply load on a struggling upstream — don't retry non-idempotent calls; keep
  `attempts` low; the overall `timeout` must exceed `attempts × perTryTimeout` or retries are cut
  off.

## Fault injection (testing resilience)

```yaml
http: [{ fault: {delay: {percentage: {value: 10}, fixedDelay: 5s},
                 abort: {percentage: {value: 5}, httpStatus: 503}}, route: [...] }]
```

Inject latency/errors to test the caller's timeouts/retries/circuit breakers — in staging, never
production. Remove after the test (a stray fault rule is a real incident).

## ServiceEntry and egress

- External dependencies (a SaaS API, a DB outside the mesh) need a `ServiceEntry` to get mTLS
  origination, telemetry, and policy — otherwise they're opaque `PassthroughCluster` traffic.
- Egress control: a `Sidecar` egress scope or an egress gateway forces external traffic through a
  controlled path (allowlist, monitoring) rather than any pod dialing the internet directly.

## Verify (read-only)

```bash
istioctl analyze -A                                  # catches subset/host/gateway mismatches
istioctl proxy-config routes <pod>                   # what Envoy actually has
istioctl x describe pod <pod>                        # effective routing/policy summary
```
