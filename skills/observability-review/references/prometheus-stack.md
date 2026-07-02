# Prometheus / Grafana stack deployment

Self-authored reference (no third-party source). Deploying and operating the
kube-prometheus-stack, the CRDs that drive scraping and alerting, storage/retention/HA
decisions, cardinality control, and Grafana provisioning. Deployment-planning: propose
values/manifests, never apply to a live cluster.

## What the stack is

`prometheus-community/kube-prometheus-stack` (Helm) bundles the Prometheus Operator + a full
monitoring stack. Components:

| Component | Role |
|---|---|
| Prometheus Operator | Reconciles CRDs → Prometheus/Alertmanager config; the reason you never hand-edit prometheus.yml |
| Prometheus | Scrape + TSDB + rule evaluation |
| Alertmanager | Routing, grouping, silencing, notification |
| Grafana | Dashboards |
| node-exporter (DaemonSet) | Node/hardware metrics (USE) |
| kube-state-metrics | Kubernetes object state (deployments, pods, PVCs…) |

The Operator's value: scrape targets and rules come from **CRDs**, not a static config file.

## The CRDs that replace prometheus.yml

| CRD | Replaces | Use |
|---|---|---|
| `ServiceMonitor` | scrape_config | Scrape endpoints behind a Service (the default pattern) |
| `PodMonitor` | scrape_config | Scrape pods without a Service |
| `PrometheusRule` | rule_files | Alerting + recording rules |
| `Alertmanager` config (`AlertmanagerConfig` CR / secret) | alertmanager.yml | Routing |
| `Probe` | blackbox scrape | Synthetic/blackbox checks |

- ✅ App teams ship a `ServiceMonitor` next to their app — self-service scraping, GitOps-managed.
- Gotcha: the Operator only picks up ServiceMonitors matching its `serviceMonitorSelector`.
  Default kube-prometheus-stack sets a release label requirement — a ServiceMonitor without the
  matching label is silently ignored. `serviceMonitorSelectorNilUsesHelmValues: false` +
  explicit selectors, or label every ServiceMonitor with the release, or the target never scrapes.
- ServiceMonitor needs a **named port** on the Service and the right `path`/`interval`; the
  annotation-based `prometheus.io/scrape` style is the *old* way and isn't read by the Operator.

## Storage, retention, remote-write

- Prometheus needs a PVC (`prometheus.prometheusSpec.storageSpec`) — default emptyDir loses all
  history on restart. Size = ingest rate × retention; monitor `prometheus_tsdb_*` for headroom.
- `retention: 15d` (time) and/or `retentionSize: 50GB` — whichever hits first. Local retention
  is for recent debugging, not long-term.
- Long-term / global view → `remoteWrite` to Thanos, Mimir, or VictoriaMetrics; Prometheus stays
  the scraper, the remote store holds history and does cross-cluster queries. Flag "we keep 1y
  in local Prometheus" as a scaling mistake.
- HA: run 2 Prometheus replicas (`replicas: 2`) scraping the same targets; dedupe happens at the
  query layer (Thanos querier / the stack's built-in). Two replicas without a dedup layer = double
  everything in graphs.

## Cardinality — the #1 way this stack falls over

Every unique label-value combination is a series; unbounded labels OOM Prometheus:

- ❌ labels with unbounded values: user IDs, request IDs, full URLs, error messages as labels.
- ✅ bound them: `path` as a route template (`/users/:id`), status class not exact code where
  possible.
- `metric_relabel_configs` on the ServiceMonitor to **drop** high-cardinality series at scrape
  time (`action: drop` on labels/metrics you don't query).
- Watch `prometheus_tsdb_head_series`; `topk(20, count by (__name__)({__name__=~".+"}))` finds
  the worst offenders (read-only). A cardinality explosion from one bad label ships as an OOM,
  not a warning.
- kube-state-metrics + high pod churn is a common source — filter namespaces/labels if not used.

## Resource sizing and self-monitoring

- Prometheus memory scales with active series (rough: ~few KB/series in head + WAL); set
  requests/limits from measured `process_resident_memory_bytes`, not guesses.
- Monitor the monitor: alert on `up == 0` for critical jobs, Prometheus/Alertmanager own
  health, `prometheus_notifications_dropped_total`, TSDB reload failures, and PVC fullness.
  A blind spot in the monitoring stack is the worst blind spot.

## Alertmanager routing (config side)

Routing tree by `severity` label (matches the `alert-rule-review` skill's routing checks):
```yaml
route:
  group_by: [alertname, namespace]        # service-level, not per-pod (page storms)
  receiver: default
  routes:
    - matchers: [severity="page"]
      receiver: pagerduty
    - matchers: [severity="ticket"]
      receiver: jira
inhibit_rules:
  - source_matchers: [alertname="NodeDown"]     # node down inhibits its pods' alerts
    target_matchers: [severity="page"]
    equal: [node]
```
`repeat_interval` 4–12h for pages; a receiver for every severity that's actually used (an unused
severity lands in `default` — verify where that goes).

## Grafana provisioning (dashboards & datasources as code)

- Datasources: provisioned via Helm values (`grafana.datasources`) or a provisioning ConfigMap —
  never clicked in the UI (lost on pod restart).
- Dashboards: the sidecar pattern — a ConfigMap labeled `grafana_dashboard: "1"` is hot-loaded;
  dashboards live in git as JSON (see `grafana-dashboards` skill for authoring/uid pinning).
- ✅ provisioned + read-only + PR workflow; ❌ UI-edited dashboards (drift or lost).
- Grafana itself: persist or fully provision (a stateless Grafana with everything provisioned is
  the clean GitOps shape — no PVC, rebuildable from git).

## GitOps & bootstrap ordering

- Whole stack in the GitOps repo. CRDs must exist before any ServiceMonitor/PrometheusRule that
  uses them → the stack (with CRDs) syncs in an early wave, app ServiceMonitors later.
- kube-prometheus-stack CRDs are large; ArgoCD may need `ServerSideApply=true` (they exceed the
  client-side apply annotation size limit) — a common first-sync failure. (Cross-ref
  `argocd-debug`.)
- CRD upgrades are a deliberate step on chart upgrades — the chart does not upgrade CRDs
  automatically; apply the new CRDs from the release first.

## Troubleshooting (read-only)

```bash
# Target down / not scraped — check Prometheus targets and the ServiceMonitor match
kubectl get servicemonitor -A
kubectl get prometheus -o jsonpath='{.items[0].spec.serviceMonitorSelector}'    # what it selects
# then Prometheus UI /targets, or:
kubectl logs -n monitoring prometheus-<sts>-0 -c prometheus | grep -i "scrape\|error"
```

| Symptom | Cause |
|---|---|
| App exposes /metrics but no data | ServiceMonitor missing the release label the Operator selects; wrong port name/path |
| Prometheus OOMKilled | Cardinality explosion — find via `topk(... count by (__name__))`; drop labels |
| History gone after restart | emptyDir instead of PVC storageSpec |
| Alerts fire but no notification | Alertmanager route/receiver mismatch on severity label; check `amtool config routes` |
| First ArgoCD sync fails on CRDs | CRD size > client-side apply limit → ServerSideApply |
| Duplicate series in graphs | 2 Prometheus replicas without a dedup/query layer |
