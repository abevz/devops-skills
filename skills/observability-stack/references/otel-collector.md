# OpenTelemetry Collector

Self-authored reference (no third-party source). Building Collector pipelines that don't OOM.
Propose config; validate with `otelcol validate`, don't hot-swap a live gateway blind.

## Pipeline model

A Collector pipeline is **receivers → processors → exporters**, one pipeline per signal
(traces / metrics / logs):

| Stage | Common ones |
|---|---|
| Receivers | `otlp` (gRPC/HTTP — the standard app ingress), `prometheus` (scrape), `filelog`, `hostmetrics`, `kafka` |
| Processors | `memory_limiter`, `batch`, `k8sattributes`, `resourcedetection`, `tail_sampling`, `transform`/`filter` |
| Exporters | `otlp`/`otlphttp`, `prometheusremotewrite`, `loki`, `debug` |

```yaml
processors:
  memory_limiter: {check_interval: 1s, limit_percentage: 80, spike_limit_percentage: 25}
  batch: {send_batch_size: 8192, timeout: 5s}
service:
  pipelines:
    traces:  {receivers: [otlp], processors: [memory_limiter, k8sattributes, batch], exporters: [otlp/tempo]}
    metrics: {receivers: [otlp, prometheus], processors: [memory_limiter, batch], exporters: [prometheusremotewrite]}
    logs:    {receivers: [otlp, filelog], processors: [memory_limiter, batch], exporters: [otlphttp/loki]}
```

- **`memory_limiter` must be first** in every pipeline — it sheds load before the Collector OOMs;
  put it ahead of `batch`. Skipping it is the #1 Collector OOM cause.
- `batch` for throughput/efficiency (fewer, bigger exports). `k8sattributes` enriches with
  pod/namespace/labels — put it before batch so downstream has the metadata.

## Topology: agent + gateway

| Tier | Deploy | Job |
|---|---|---|
| **Agent** | DaemonSet (per node) | Collect node-local: `filelog`, `hostmetrics`, receive app OTLP on localhost; forward to gateway |
| **Gateway** | Deployment (scaled) | Central: aggregation, **tail sampling**, fan-out to backends, egress auth |

- The agent+gateway pattern is the scalable default. Small setups can run agent-only (export
  straight to backends); add the gateway when you need central sampling/aggregation/egress control.
- **Tail sampling only works at the gateway**, never the agent: it must see **all spans of a
  trace** to decide (keep errors/slow traces), so all spans of a trace must land on one gateway
  instance — load-balance by trace ID (`loadbalancing` exporter agent→gateway) or tail sampling
  silently makes wrong keep/drop decisions.
- `memory_limiter` on the gateway especially — it's the aggregation choke point.

## What it replaces / unifies

- Replaces Promtail (logs via `filelog` → Loki), can replace Prometheus scraping (`prometheus`
  receiver → `prometheusremotewrite`), and ingests app OTLP — one pipeline for all three signals.
- OTLP is the wire protocol: app SDK → Collector → backend. Standardizing on OTLP keeps backends
  swappable.
- Migration: run the Collector alongside existing agents, cut over one signal/source at a time,
  compare, then retire the old agent.

## Gotchas

- No `memory_limiter` → OOM under burst. Tail sampling at the agent → wrong decisions. Unbounded
  `filelog` on chatty nodes → the Collector becomes the load. `prometheusremotewrite` inherits all
  the remote-write cardinality/cost concerns (drop labels in a `transform`/`filter` processor).
- Persistent queue (`sending_queue` + file storage) so a backend blip doesn't drop data — but
  size the disk.

## Verify (read-only)

```bash
otelcol validate --config config.yaml       # config sanity, no live change
# Collector self-metrics: otelcol_processor_refused_*, otelcol_exporter_send_failed_*,
#   otelcol_processor_batch_batch_send_size — watch refused/dropped
```
