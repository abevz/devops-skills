# Prometheus HA, sharding, remote-write, sizing

Self-authored reference (no third-party source). Scaling Prometheus itself. (Base kube-prometheus-
stack install, ServiceMonitor wiring, cardinality basics live in `observability-review/references/
prometheus-stack.md` — this is the scaling layer on top.) Propose config; don't restart/reshape a
live TSDB.

## Sizing — active series is everything

- Prometheus memory ≈ **active series × per-series overhead** (roughly a few KB/series incl.
  indexes) plus scrape/query working set. **Cardinality, not scrape count, drives OOM.**
- Get the number before sizing: `prometheus_tsdb_head_series` (current active series),
  `prometheus_tsdb_head_chunks`, `scrape_samples_scraped`. `topk` on
  `count by (__name__)({__name__=~".+"})` finds the metric blowing up series.
- WAL replays on restart into memory — a huge head block means slow, memory-spiky restarts.
- Recording rules precompute expensive/repeated queries (dashboards, alerts) into new series —
  cheaper at query time, but each rule adds series; use for genuinely hot queries, not everything.

## HA (no gaps when one dies)

- Run **2+ identical replicas** scraping the same targets (Prometheus Operator `spec.replicas: 2`).
  They're independent TSDBs — samples differ slightly (scrape timing), so you can't naively
  round-robin them or graphs jump.
- Dedup happens at the **query layer**: Thanos Querier / Mimir dedup by an HA-replica label
  (`prometheus_replica` / external label). Without a dedup-aware query layer, HA means "manual
  failover" and visibly discontinuous graphs on switch.
- HA doubles ingest cost (two scrapes of everything) — it buys no-gap availability, not more
  capacity. It's orthogonal to sharding.

## Sharding (series don't fit one instance)

- When one Prometheus can't hold the cluster's series, **shard by target**: Prometheus Operator
  `spec.shards: N` creates N StatefulSets, each scraping a `hashmod`-selected subset of targets.
- Query across shards needs a global view (Thanos/Mimir) — sharding without an aggregating query
  layer fragments your data.
- Shard by *target*, so all series from one target land on one shard (a single huge target still
  overloads its shard — sharding doesn't split within a target).
- Reach for sharding only after cutting cardinality and using recording rules — it's the "still
  too big" answer, not the first one.

## remote-write (centralize / long-term)

```yaml
# ship samples to an LTS (Mimir/Thanos Receive/Cortex); tune the queue
remoteWrite:
  - url: https://mimir/api/v1/push
    queueConfig: {capacity: 10000, maxShards: 50, maxSamplesPerSend: 2000}
    writeRelabelConfigs: [ ... ]        # drop noisy series BEFORE they leave (cost control)
```

- remote-write is **CPU/memory heavy on the sender** (serialization + queues) — budget for it;
  a badly-tuned queue backs up and drops samples (`prometheus_remote_storage_samples_failed_total`).
- `writeRelabelConfigs` to **drop** high-cardinality series before shipping is the main cost lever
  for push-based LTS — filter at the edge, don't pay to store noise.
- Prometheus stays the scraper/source of truth (pull); remote-write adds a push tail to central
  storage. Local retention can then be short (a few days) with LTS holding the long tail.

## Retention and federation

- Local `--storage.tsdb.retention.time` (e.g. 15d) / `retentionSize` — keep local short once LTS
  exists; long local retention just bloats the instance.
- **Avoid `/federate` for scale** — hierarchical federation (a top Prometheus pulling aggregates)
  is legacy and brittle for large setups; use remote-write + Thanos/Mimir for global/long-term.
  `/federate` is fine only for pulling a few pre-aggregated series.

## Verify (read-only)

```bash
promtool check rules rules.yaml
# Prometheus HTTP: /api/v1/status/tsdb (top series by name/label), /api/v1/status/runtimeinfo
# key metrics: prometheus_tsdb_head_series, prometheus_remote_storage_samples_pending,
#              prometheus_target_scrapes_exceeded_sample_limit_total
```
