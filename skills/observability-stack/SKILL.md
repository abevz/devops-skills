---
name: observability-stack
description: Use when deploying, scaling, or operating the observability stack itself — Prometheus HA/sharding/remote-write, long-term storage (Thanos vs Mimir), OpenTelemetry Collector pipelines, and log/trace backends (Loki, Tempo). Mention "prometheus HA", "prometheus sharding", "thanos", "mimir", "remote write", "otel collector", "loki", "tempo", "long term metrics" as triggers.
license: MIT
compatibility: Works with Claude Code, Codex-style agents, CodeWhale, OpenCode, and other agents that support Agent Skills-style instructions.
---

# Observability stack

## TL;DR checklist

- [ ] Identify the deployed telemetry components and target area.
- [ ] Measure active series, traffic volume, retention, and current bottleneck.
- [ ] Choose scale or storage changes against the measured problem.

## Key read-only checks

- Read Prometheus status, target and series counts, retention, remote-write, and object-storage configuration.

## Common pitfalls

- Do not recommend long-term storage before measuring cardinality and retention needs.

## Agent procedure

Follow the [Workflow](#workflow), [Safety rules](#safety-rules), and [Quality checklist](#quality-checklist) below.

## Quick references

- [references/prometheus-ha-scaling.md](references/prometheus-ha-scaling.md)
- [references/long-term-storage.md](references/long-term-storage.md)

Last verified: unverified

## When to use

Use when standing up or scaling the observability *platform*: Prometheus HA and sharding,
remote-write, long-term metric storage (Thanos vs Mimir), OpenTelemetry Collector pipelines, and
log/trace backends (Loki, Tempo). For *what to instrument and which signals/SLOs matter* use
`observability-review`; for the base kube-prometheus-stack install, ServiceMonitor wiring, and
cardinality basics see `observability-review/references/prometheus-stack.md`; for alert rule
quality use `alert-rule-review`; for dashboards use `grafana-dashboards`. This skill is the
platform/operator view of the stack itself.

## Goal

An observability stack that holds the series/logs/traces you actually produce without falling
over: Prometheus HA'd and sharded to fit its cardinality, long-term storage chosen deliberately
(pull-native Thanos vs push-based Mimir), a Collector pipeline that won't OOM, and log/trace
backends whose cost model you understand (label cardinality, no-index traces) — all sized from
real series/volume, not guessed.

## Workflow

1. **Scope to an area** and load its reference:

   | Area | Reference |
   |---|---|
   | Prometheus HA, sharding, remote-write, retention/WAL, recording rules, sizing | `references/prometheus-ha-scaling.md` |
   | Long-term storage: Thanos vs Mimir, object storage, compaction/downsampling, global query | `references/long-term-storage.md` |
   | OpenTelemetry Collector: receivers/processors/exporters, agent+gateway topology, tail sampling | `references/otel-collector.md` |
   | Logs & traces backends: Loki (label cardinality), Tempo (no-index traces), correlation | `references/logs-traces.md` |

2. **Confirm the baseline** — what's deployed today (kube-prometheus-stack? bare Prometheus?),
   **active series count** (drives everything), scrape/log/trace volume, current retention, and
   whether there's already object storage. The single most important number is active series.
3. **Size from real numbers** — Prometheus memory ≈ active series × per-series bytes; cardinality
   is the OOM driver, not scrape count. Get the number (`prometheus_tsdb_head_series`) before
   recommending replicas/shards/LTS.
4. **Don't over-build** — a single Prometheus with 15d retention is fine for a small cluster.
   HA (2 replicas), sharding, and Thanos/Mimir are answers to specific pressures (need for no
   gaps on restart, series that don't fit one instance, long retention, multi-cluster view) —
   recommend each against its trigger, not reflexively.
5. **Cardinality is a platform budget** — every new label multiplies series; the stack's cost and
   stability are governed by it. Treat runaway cardinality as the primary failure mode across
   Prometheus, Loki (labels), and metrics derived from traces.
6. **GitOps the stack** — Operator CRs, Collector config, Helm values, recording/alerting rules
   are declarative and belong in git; big CRDs may need ServerSideApply (see prometheus-stack.md).
7. **Verify read-only** — `promtool`, Prometheus `/status/tsdb` and `/api/v1/status/*`, Thanos/
   Mimir status endpoints, `otelcol` config validate; never restart ingesters/compactors or
   mutate storage as "investigation."

## References

- `references/prometheus-ha-scaling.md`, `references/long-term-storage.md`,
  `references/otel-collector.md`, `references/logs-traces.md` — one per area, with decision
  tables, config examples, prerequisites, and gotchas.

## Safety rules

- Design/operate-planning skill: do not apply Operator CRs, restart ingesters/store-gateways/
  compactors, delete TSDB blocks, or change retention on a live stack — propose config.
- The Thanos **Compactor must be a singleton per bucket** — never propose running two against one
  object store (it corrupts blocks); treat as a hard rule.
- Retention/compaction/block-deletion changes are destructive to history — explicit confirmation,
  never automatic.
- Cardinality changes (adding labels, relabel rules) can multiply cost/series and OOM the stack —
  frame as budgeted, reviewed changes, and prefer dropping labels over adding.
- Object-storage credentials and remote-write auth are secrets — reference a secrets mechanism
  (`secrets-management`), never inline or echo.

## Output format

```
## Area & baseline (what's deployed, active series, volume, retention, object storage?)
## Recommendation (what to deploy/scale, and explicitly what NOT to over-build)
## Prerequisites & blast radius (object storage, resources, singleton constraints)
## Proposed config (Operator CRs / Collector pipelines / Helm values — for review)
## Rollout & rollback
## Verification (read-only status/promtool checks, cardinality budget)
```

## Quality checklist

- [ ] The relevant area reference was consulted, not reasoned from memory
- [ ] Recommendation is sized from actual active-series / volume, not guessed
- [ ] HA/sharding/LTS each recommended against a real trigger, not reflexively
- [ ] Thanos Compactor singleton-per-bucket respected; no double-compaction proposed
- [ ] Cardinality budget considered (Prometheus series, Loki labels, span-derived metrics)
- [ ] Storage credentials referenced via a secrets mechanism, never inline
- [ ] Nothing was restarted/deleted/retention-changed on a live stack
