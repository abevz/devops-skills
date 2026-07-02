# Istio telemetry

Self-authored reference (no third-party source). Metrics, access logs, tracing via the Telemetry
API, plus Kiali and cardinality control. Propose config; don't apply.

## Telemetry API (the modern way)

The `Telemetry` CRD configures metrics/logs/tracing per mesh/namespace/workload — replacing the
old EnvoyFilter hacks and pod annotations:

```yaml
kind: Telemetry
metadata: {name: mesh-default, namespace: istio-system}   # mesh-wide when in the root ns
spec:
  metrics: [{providers: [{name: prometheus}]}]
  accessLogging: [{providers: [{name: envoy}]}]
  tracing: [{providers: [{name: otel}], randomSamplingPercentage: 1.0}]
```

Scope it: mesh-wide defaults in the root namespace, override per namespace/workload with a
`selector`. ❌ new EnvoyFilter-based telemetry config — use the Telemetry API.

## Metrics → Prometheus

- Standard Istio metrics: `istio_requests_total`, `istio_request_duration_milliseconds`,
  `istio_tcp_*` — the RED signals for every service, emitted by the proxies.
- Scrape via ServiceMonitors/PodMonitors (mind the kube-prometheus-stack selector-label trap —
  `observability-review/prometheus-stack.md`).
- **Cardinality is the trap**: default labels include `source_workload`, `destination_workload`,
  `response_code`, etc. In a big mesh this is a lot of series; adding custom dimensions (paths,
  headers) via `metrics.overrides` can explode it. Drop unneeded dimensions with `tagOverrides`/
  `disabled`; watch `prometheus_tsdb_head_series` (cross-ref cardinality section in the
  prometheus-stack reference).

## Access logs

- Envoy access logs per request — invaluable for `istio-debug` (the response-flag taxonomy comes
  from here). Enable via Telemetry `accessLogging`; they're **off by default** in some profiles.
- Cost: one log line per request — for high-RPS services, sample or scope to what you'd
  investigate, and ship to Loki/ELK, don't just leave on stdout.

## Distributed tracing

- Providers: OpenTelemetry (preferred), Zipkin, Jaeger. Set `randomSamplingPercentage` low
  (1–5%) in prod — 100% sampling is expensive and rarely needed.
- **Context propagation is on the app, not just Istio**: Istio creates spans, but the app **must
  forward the trace headers** (B3 / W3C `traceparent`) from incoming to outgoing requests, or the
  trace breaks at each hop. This is the #1 "tracing shows disconnected spans" cause — a mesh
  can't propagate context through your app for you.

## Kiali

- Service graph + config validation UI over the Istio metrics + config. Read-only view of
  traffic flow, mTLS status per edge, and `istioctl analyze`-style config problems visually.
- Needs Prometheus (for the graph) and access to Istio config; deploy it read-only where possible
  (it can also apply config — restrict that in shared clusters).

## What to alert vs dashboard

- Alert (via `alert-rule-review`): request error rate / p99 latency per service from
  `istio_requests_total`/`istio_request_duration`; gateway 5xx; cert expiry approaching; proxies
  not SYNCED.
- Dashboard: the standard Istio Grafana dashboards (mesh/service/workload) — provision as code
  (`grafana-dashboards`); Kiali for topology.

## Verify (read-only)

```bash
istioctl analyze -A
kubectl get telemetry -A
istioctl proxy-config bootstrap <pod> | grep -i -A3 tracing    # effective tracing config
```
