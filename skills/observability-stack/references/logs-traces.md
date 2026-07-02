# Log and trace backends: Loki and Tempo

Self-authored reference (no third-party source). Operating log/trace storage and correlating the
three signals. Propose config; don't reshape live storage.

## Loki — labels are the cost, not content

- Loki indexes **labels**, not log content (unlike Elasticsearch). Queries filter by label to a
  stream, then grep (LogQL) the content. This makes it cheap — *if* you keep label cardinality low.
- ⚠️ **The cardinality trap**: putting a high-cardinality value in a **label** (request_id,
  trace_id, user_id, pod IP) explodes the number of streams and destroys Loki. Keep those in the
  **log line**, filter with LogQL; labels are for low-cardinality dimensions (namespace, app,
  level, cluster). This is the single most important Loki rule and the #1 way people melt it.
- Deploy modes:

  | Mode | Use |
  |---|---|
  | Monolithic (single binary) | Small / dev |
  | **Simple Scalable** (read / write / backend targets) | The production sweet spot — scale reads and writes independently without full microservices |
  | Microservices | Very large, need per-component scaling |

- Chunks live in **object storage** (same durability story as metrics LTS). Retention via the
  compactor + per-tenant limits.
- Ingest via the OTel Collector (`filelog`/`otlphttp`) or Promtail (legacy — prefer the Collector).

## Tempo — traces, no index, cheap

- Tempo stores traces in **object storage with no big index** — that's why it's cheap at scale
  (vs Elasticsearch-backed tracing). The tradeoff: you **find traces by ID or TraceQL**, and you
  usually *discover* the ID from elsewhere (an exemplar on a metric, a trace_id in a log).
- So Tempo isn't standalone: it pairs with metrics (exemplars) and logs (trace_id field) to be
  navigable. Design that correlation in, or traces are a write-only black hole.
- **Metrics-generator**: Tempo can derive RED metrics (rate/errors/duration) and service-graph
  metrics from spans and remote-write them to Prometheus/Mimir — span-derived metrics carry
  cardinality (per service pair), so budget them like any other series.
- Sampling: head sampling in the SDK (cheap, dumb) vs **tail sampling in the OTel gateway** (keep
  errors/slow, drop boring) — see `otel-collector.md`. Store what's worth storing.

## Correlation — the reason to run a coherent stack

The payoff of a coherent stack (Prometheus/Mimir + Loki + Tempo in Grafana, or an equivalent) is
one-click pivots:

- **exemplars**: metric spike → jump to an example trace (needs exemplar storage on + span-metrics)
- **trace → logs**: from a span, jump to that pod/time-window's logs (shared labels/trace_id)
- **logs → trace**: a `trace_id` in a structured log line links straight to the trace

Getting this requires consistent metadata across signals (the `k8sattributes` processor,
consistent resource labels) — it doesn't happen for free; it's a design choice.

## Jaeger note

Jaeger is the mature alternative for traces; Tempo wins on cost at scale (no index) and native
Grafana correlation. Choose Jaeger if you're already invested in it or need its specific UI/
features; new Grafana-stack deploys usually pick Tempo.

## Gotchas recap

- Loki: high-cardinality **label** = outage. Keep IDs in the line.
- Tempo: no discovery path (exemplars/log trace_ids) = unusable traces.
- Both: object storage credentials are secrets (`secrets-management`); retention owned by the
  compactor, not a bucket lifecycle rule that deletes underneath it.

## Verify (read-only)

```bash
logcli query '{namespace="payments"} |= "error"'   # LogQL, read-only
# Loki: /ready, /metrics (loki_ingester_streams — stream count = cardinality health)
# Tempo: /ready, /status; metrics-generator series count if enabled
```
