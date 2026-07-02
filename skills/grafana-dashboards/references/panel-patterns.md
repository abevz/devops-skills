# Grafana panel and dashboard-as-code patterns

Self-authored reference (no third-party source). Concrete query/panel/provisioning patterns
behind the SKILL.md workflow.

## Top-row verdict panels — queries that answer "is it healthy"

| Panel | Query pattern | Thresholds |
|---|---|---|
| Availability (stat) | `sum(rate(http_requests_total{code!~"5.."}[$__rate_interval])) / sum(rate(http_requests_total[$__rate_interval]))` | green ≥ SLO, red < SLO — same numbers as the alert |
| Error rate (stat/timeseries) | `sum(rate(http_requests_total{code=~"5.."}[$__rate_interval]))` — absolute rps, plus the ratio | thresholds mirror alert rule exactly |
| p95/p99 latency | `histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[$__rate_interval])) by (le))` — `by (le)` mandatory | unit: seconds (s), not ms-labeled-as-s |
| Saturation | workload-specific: queue depth, connection-pool %, CPU throttle ratio `rate(container_cpu_cfs_throttled_periods_total[...]) / rate(container_cpu_cfs_periods_total[...])` | amber at 70–80% |
| Error budget remaining (SLO boards) | `1 - (bad/total over $__range)` vs budget | gauge, red below 0 |

Always `$__rate_interval` in Grafana queries, not a hardcoded `[5m]` — it adapts to the panel's
resolution and never under-samples the scrape interval.

## Template variables

```
datasource:  type=datasource  → makes the board portable across clusters
namespace:   label_values(kube_pod_info, namespace)
service:     label_values(http_requests_total{namespace="$namespace"}, service)   # chained
pod:         label_values(..., pod)  + multi-value + includeAll
```

- Chain variables (each `label_values` filtered by the previous) or selections mix envs.
- `All` on a variable used in `by ($var)` legends explodes series count — allow `All` only where
  panels aggregate; set a custom `all_value` of `.*` and use `=~"$var"` matchers.
- Repeat-by-variable rows beat 10 hand-copied per-service rows — copies drift.

## Legend, unit, and axis discipline

- Legend `{{pod}}` / `{{ code }}` — never the raw expression (default legend = unreadable wall).
- Unit set per panel: `s`, `ms`, `reqps`, `percentunit` (0–1) vs `percent` (0–100) — mixing
  these two is the most common "dashboard lies" bug.
- `min=0` on rate/latency axes unless there's a reason; auto-scaled Y makes a 0.1% wiggle look
  like an outage.
- One meaning per color: red only for bad. Grafana's default palette cycles red onto arbitrary
  series — pin overrides on panels where red would mislead.

## Annotations that make graphs diagnosable

```json
{ "name": "deploys", "datasource": "prometheus",
  "expr": "changes(kube_deployment_status_observed_generation{namespace=~\"$namespace\"}[2m]) > 0",
  "titleFormat": "deploy: {{deployment}}" }
```

Deploy + incident annotations turn "latency jumped at 14:32" into "latency jumped at the 14:32
deploy of X" without leaving the graph. Alert-state annotations (native in unified alerting)
belong on the panels whose thresholds mirror those alerts.

## Dashboards as code — provisioning shapes

| Approach | Mechanics | Fit |
|---|---|---|
| Sidecar ConfigMaps (kube-prometheus-stack) | ConfigMap labeled `grafana_dashboard: "1"`, sidecar hot-loads JSON | GitOps-native default for k8s |
| Grafana Operator CRDs | `GrafanaDashboard` CR referencing JSON/jsonnet/URL | When already running the operator |
| File provisioning | `providers.yaml` + JSON dir mounted into the image/volume | VMs, docker-compose (homelab monitoring stacks) |
| Terraform `grafana_dashboard` | JSON in TF state, API-applied | When Grafana config is already Terraform-managed |

Rules regardless of shape:
- Pin `uid` in the JSON — links and alert annotations reference dashboards by uid; re-provision
  without one generates a new uid and breaks every bookmark/runbook link.
- Export via "Share → Export → save JSON" strips runtime noise, but still review the diff:
  `version`, `id`, and current variable selections churn — commit with `id: null`.
- UI-edited provisioned dashboards can't be saved (read-only) or drift (if editable) — pick
  read-only provisioning + PR workflow, mirroring the GitOps stance everywhere else.
- One `folder` per team/domain in provisioning config; flat hundreds-of-boards Grafana is
  where dashboards go to be lost.

## Review quick hits (existing dashboards)

- Panels with `No data` for 30+ days → metric renamed/dead → delete or fix the query.
- Two panels answering the same question with different queries → they will disagree during an
  incident; keep the one matching the alert's expression.
- p50-only latency panels → p95/p99 is where incidents live; median hides tail pain.
- `[1m]` windows hardcoded on 30–60s scrape intervals → gappy graphs → `$__rate_interval`.
- Threshold lines on graphs that don't match the current alert rule values → update together,
  ideally both generated from the same recording rule / constant.
