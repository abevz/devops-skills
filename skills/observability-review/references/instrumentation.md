# Kubernetes Instrumentation Review Reference

Distilled from LukasNiessen/kubernetes-skill (`references/observability.md`, plus observability-relevant material from `deployment-patterns.md` and `daemonset-operator-patterns.md`) during third-party review.

## Probes are the observability foundation

Liveness/readiness/startup probes are the most basic telemetry: they tell Kubernetes whether the app is alive, ready, and initialized. Without correct probes, no amount of metrics or logging prevents cascading failures — review probes before dashboards. Probes should hit dedicated `/healthz/live` and `/healthz/ready` endpoints, never the main API path (health checks under load then amplify the outage).

## Prometheus scraping: annotations vs ServiceMonitor

| Pattern | When | Key check |
|---|---|---|
| `prometheus.io/*` annotations | No prometheus-operator | Annotations must be on the **Pod template** `metadata.annotations`, not Deployment metadata |
| `ServiceMonitor` CRD | prometheus-operator installed | ServiceMonitor label (e.g. `release: kube-prometheus-stack`) must match the Prometheus operator's selector, and `endpoints[].port` must match the **Service port name** (not number) |
| `PodMonitor` CRD | No Service exists (e.g. CronJob metrics) | Same selector rules, targets pods directly |

```yaml
# annotations pattern — on pod template
template:
  metadata:
    annotations:
      prometheus.io/scrape: "true"
      prometheus.io/port: "9090"
      prometheus.io/path: "/metrics"
```

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels: {release: kube-prometheus-stack}   # must match operator selector
spec:
  selector: {matchLabels: {app: order-service}}
  endpoints:
    - port: metrics          # Service port NAME
      interval: 30s
      path: /metrics
```

The metrics port must also be declared in the container `ports` list (named, e.g. `metrics: 9090`) and match the annotation/ServiceMonitor value — a mismatch scrapes nothing silently.

## Hardened instrumented Deployment (distilled shape)

Checklist form of the source's reference example:
- Two named container ports: `http: 8080`, `metrics: 9090`.
- Scrape annotations on pod template (or ServiceMonitor alongside).
- `automountServiceAccountToken: false`; pod securityContext `runAsNonRoot`, non-zero `runAsUser/runAsGroup`, `seccompProfile: RuntimeDefault`; container `allowPrivilegeEscalation: false`, `readOnlyRootFilesystem: true`, `capabilities.drop: [ALL]`.
- Requests and limits on the app container **and on every telemetry sidecar** (OTel Collector, Fluent Bit) — an unbounded sidecar starves the workload it observes. Source baselines: sidecars at `requests: {cpu: 50m, memory: 64Mi}, limits: {memory: 128Mi}`.
- Readiness + liveness probes on dedicated endpoints.

## Metric naming, labels, buckets

RED minimum per request-driven service:

| Signal | Metric | Type |
|---|---|---|
| Rate | `http_requests_total` | counter |
| Errors | `http_requests_total{status=~"5.."}` or dedicated error counter | counter |
| Duration | `http_request_duration_seconds` | histogram |

Resource-oriented services (queues, DBs) add saturation: queue depth, connection-pool usage, disk I/O utilization.

- Histogram buckets must align to SLO thresholds, not library defaults — e.g. a fast API: `le=0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0, +Inf`. If the SLO is "p99 < 100ms" and there's no bucket at 0.1, the SLO is unmeasurable.
- Counter names end `_total`; durations in base seconds (`_seconds`); flag high-cardinality labels (user IDs, full paths).

## Alerts vs dashboards

- Alerts fire on **symptoms** (what users experience: error rate, latency percentiles), never causes (pod restarted, CPU high) — causes belong on dashboards for diagnosis.
- Every alert needs `for:` (hysteresis), a `severity` label, and a `runbook_url` annotation. Alerts without runbooks are noise.

```yaml
- alert: HighErrorRate
  expr: |
    sum(rate(http_requests_total{job="order-service",status=~"5.."}[5m]))
    / sum(rate(http_requests_total{job="order-service"}[5m])) > 0.05
  for: 5m
  labels: {severity: critical}
  annotations:
    summary: "Order service error rate above 5%"
    runbook_url: "https://wiki.example.com/runbooks/order-service-errors"
- alert: HighLatencyP99
  expr: histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket{job="order-service"}[5m])) by (le)) > 2.0
  for: 10m
  labels: {severity: warning}
```

Deploy markers: POST to the Grafana annotations API (`/api/annotations`, tags like `deployment`) from CI post-deploy, so metric shifts correlate with releases.

## Logging patterns

- Structured JSON to stdout/stderr only. ❌ file logging inside the container — defeats node-level collection and fills the writable layer (fatal with `readOnlyRootFilesystem`).
- Standard fields: `timestamp`, `level`, `msg`; include `trace_id`/`span_id` for log↔trace correlation. Error-level to stderr, rest to stdout (some collectors distinguish).
- Never log secrets, tokens, PII, or full request bodies.

```json
{"timestamp":"2025-03-15T10:23:45Z","level":"error","msg":"payment failed","trace_id":"abc123","order_id":"ord-789","error":"timeout after 5s"}
```

Collection topology:

| Pattern | Use |
|---|---|
| Node-level DaemonSet (Fluent Bit/Vector reading `/var/log/containers/`) | Default for everything |
| Sidecar collector | Only for per-pod log transformation or apps that cannot log to stdout |

DaemonSet collector review points: `hostPath` mounts `readOnly: true`, `hostPath.type` set, custom PriorityClass (not `system-node-critical`), specific tolerations only, tight requests (they multiply by node count).

## Tracing

- OTel Operator auto-instrumentation via annotation: `instrumentation.opentelemetry.io/inject-java: "true"` (also `-python`, `-nodejs`).
- OTel Collector sidecar: OTLP receivers on 4317 (gRPC) / 4318 (HTTP); hardened securityContext and resource limits like any container.
- W3C Trace Context (`traceparent`) must propagate across every service boundary — without propagation traces fragment and are useless.

## Review checklist (from source)

- [ ] Scrape annotations on pod template, not Deployment metadata
- [ ] Metrics port declared in `ports` and consistent with annotation/ServiceMonitor
- [ ] Structured JSON to stdout; no file logging
- [ ] Trace propagation configured (annotation or SDK), trace_id present in logs
- [ ] Alerts symptom-based; every alert has runbook_url; `for:` set
- [ ] Histogram buckets match SLO thresholds
- [ ] Requests/limits on all telemetry sidecars
