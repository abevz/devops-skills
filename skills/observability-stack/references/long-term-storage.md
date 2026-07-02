# Long-term metric storage: Thanos vs Mimir

Self-authored reference (no third-party source). Choosing and operating global/long-term metric
storage. Propose config; block deletion/compaction is destructive.

## The decision

| | Thanos | Mimir (Grafana) |
|---|---|---|
| Model | **Pull-native**: sidecar uploads Prometheus TSDB blocks to object storage | **Push-based**: Prometheus `remote_write` in |
| Prometheus role | Stays source of truth; Thanos adds global view + LTS | Becomes a thin forwarder; Mimir owns storage |
| Multi-tenancy | Bolt-on | **Native** (`X-Scope-OrgID`), first-class |
| Ops weight | Lighter to start (sidecar), more moving parts at scale | Heavier system, but one coherent horizontally-scalable store |
| Best for | Keep Prometheus-centric, add global query + cheap LTS | Large, multi-tenant, central metrics platform |

- Cortex is Mimir's predecessor — **new deploys use Mimir**, not Cortex.
- Both store TSDB **blocks in object storage** (S3/GCS/Azure) — object storage is a hard prereq;
  its availability/durability is now your metrics durability.

## Thanos components (and the footgun)

| Component | Role |
|---|---|
| Sidecar | Runs beside each Prometheus; uploads blocks to the bucket, serves recent via StoreAPI |
| Store Gateway | Serves historical blocks from object storage to queriers |
| **Compactor** | Compacts + downsamples (5m, 1h) blocks in the bucket |
| Querier | Fans out to sidecars + store gateways, **dedups HA replicas** |
| Query Frontend | Splits/caches queries (put in front of Querier for big dashboards) |
| Ruler | Evaluates recording/alerting rules against the global view |

- ⚠️ **The Compactor MUST be a singleton per bucket.** Two compactors on one object store corrupt
  blocks — no HA, no two replicas, ensure a single instance (leader/`replicas: 1`). This is the
  classic Thanos production incident.
- Downsampling (5m/1h) makes long-range queries cheap; raw is kept for the retention you set per
  resolution (`--retention.resolution-raw/5m/1h`).
- Querier dedup needs the HA `replica` external label set on each Prometheus (so it knows which
  series are duplicates).

## Mimir components

- `distributor` (receives remote-write, dedups HA at ingest), `ingester` (holds recent series in
  memory, **replication factor 3** for durability before flush to blocks), `store-gateway`,
  `querier`, `query-frontend`, `compactor`, `ruler`.
- Deploy modes: **monolithic** (one binary, all targets — small/medium) vs **microservices**
  (scale each component — large). `mimir-distributed` Helm chart.
- Ingesters are stateful and the sensitive part: rolling them needs care (they hold un-flushed
  data; RF3 + proper `-ingester.ring` handling avoids data loss on restart).
- Native multi-tenancy: every write/read carries `X-Scope-OrgID`; per-tenant limits
  (series/ingestion-rate) are the guardrail against one tenant OOMing the cluster.

## Choosing shortcut

- Keep Prometheus central, want global query + cheap long retention, minimal push infra →
  **Thanos** (sidecar model). Accept the compactor-singleton rule + more components at scale.
- Want a push-based, multi-tenant, horizontally-scalable central metrics store, willing to run a
  bigger system → **Mimir**. Prometheus becomes a forwarder.

## Common to both

- Object storage lifecycle: don't set a bucket lifecycle rule that deletes blocks out from under
  the compactor/retention — let the tool own retention, not the bucket policy.
- Bucket credentials are secrets (IRSA/Workload Identity preferred over static keys — see
  `secrets-management`).
- Query Frontend + caching is what makes big multi-month dashboards usable — add it before blaming
  the store for slow queries.

## Verify (read-only)

```bash
thanos tools bucket inspect --objstore.config-file=... # blocks in the bucket (read-only)
# Thanos: /api/v1/status, compactor /loaded; Mimir: /ready, /config, ring status pages
```
