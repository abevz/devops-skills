# PromQL alerting patterns and pitfalls

Self-authored reference (no third-party source). Concrete expressions, math, and failure modes
for reviewing or writing Prometheus alert rules.

## Multi-window burn-rate alerts (SLO-based paging)

For an SLO with error budget `1 - SLO` (e.g. 99.9% → budget 0.1%), burn rate = observed error
ratio ÷ budget. The standard Google SRE multiwindow pairs:

| Alert | Burn rate | Long window | Short window | Budget consumed | Severity |
|---|---|---|---|---|---|
| Fast burn | 14.4× | 1h | 5m | 2% of 30d budget in 1h | page |
| Mid burn | 6× | 6h | 30m | 5% in 6h | page |
| Slow burn | 1× | 3d | 6h | 10% in 3d | ticket |

Both windows must fire together (short window stops alerting the moment the burn stops —
prevents paging on an already-resolved spike):

```yaml
expr: (
    sum(rate(http_requests_total{code=~"5.."}[1h])) / sum(rate(http_requests_total[1h])) > 14.4 * 0.001
  and
    sum(rate(http_requests_total{code=~"5.."}[5m])) / sum(rate(http_requests_total[5m])) > 14.4 * 0.001
)
```

When burn-rate is overkill: no defined SLO, very low traffic (ratio of tiny numbers is noise —
gate with `sum(rate(http_requests_total[1h])) > 1` minimum-traffic condition), or batch systems
(alert on freshness/lag instead).

## Absence blindness — the paired-rule pattern

Every threshold alert silently dies when its target disappears. Critical services need the pair:

```yaml
- alert: HighErrorRate
  expr: sum(rate(errors_total[5m])) by (job) / sum(rate(requests_total[5m])) by (job) > 0.05
- alert: TargetAbsent
  expr: absent(up{job="payments"}) or up{job="payments"} == 0
  for: 5m
```

- ❌ `absent(metric{label="x"})` with a label the metric never had — always fires or never fires;
  test the selector against real data first.
- ❌ Relying on `up` alone for pushgateway/remote-write sources — `up` tracks scrapes, not the
  actual producer; use `time() - max(metric_push_timestamp) > 300` freshness instead.

## rate/increase pitfalls

- ❌ `rate(x[1m])` on a 30s scrape interval — 2 samples, one lost scrape = empty result. Rule:
  window ≥ 4× scrape interval; in Grafana use `$__rate_interval`, in rules just size it.
- ❌ `rate(x[$window]) * $window_seconds` to "count events" — use `increase()` and accept it's
  an extrapolated estimate, not an integer; don't alert on `increase() == 1` exactly.
- ❌ `sum(rate(...))` vs `rate(sum(...))` — the second is wrong: sum first destroys counter-reset
  detection. Always `sum(rate(x[5m]))`, never `rate(sum(x)[5m])`.
- ❌ Comparing raw counters (`errors_total > 100`) — resets on restart make it meaningless.
- Histogram latency: `histogram_quantile(0.99, sum(rate(http_duration_bucket[5m])) by (le))` —
  the `by (le)` is mandatory; forgetting it silently returns garbage quantiles. Quantile
  accuracy is bounded by bucket layout — a p99 alert threshold *between* two bucket bounds
  can't be measured; align thresholds to bucket boundaries.

## for:, keep_firing_for, and flap control

- `for:` must exceed at least 2 evaluation intervals AND survive a single scrape gap. Under 2m
  on a 1m eval = single-blip pages.
- `keep_firing_for` (Prometheus 2.42+) prevents resolve/refire flapping on metrics hovering at
  the threshold — cheaper than hysteresis hacks.
- True hysteresis when needed: alert fires at X, clears at Y<X via
  `metric > X or (metric > Y and ALERTS{alertname="Me", alertstate="firing"})` — document it;
  it confuses readers.

## Label and cardinality review

- Alert `expr` aggregations must keep the labels the notification template uses:
  `sum by (namespace, service)` if the annotation says `{{ $labels.service }}` — a dropped
  label renders as empty string in the page.
- ❌ `by (pod)` / `by (instance)` on paging alerts for replicated services — one incident = N
  pages. Aggregate to service level; keep pod detail for the dashboard.
- ❌ Alert selectors matching high-cardinality label values (`path=~".*"` on unbounded paths) —
  rule evaluation cost scales with series; check `prometheus_rule_group_last_duration_seconds`.

## Recording rules

Name convention `level:metric:operations` (e.g. `namespace:http_errors:rate5m`). Promote an
expression to a recording rule when: used by ≥2 alerts/panels, or evaluation >1s, or it needs a
long range (burn-rate 3d windows over raw data are expensive). Alert rules should then reference
the recorded series — divergence between an alert's inline expr and the dashboard's recorded
version is a classic "dashboard green, alert firing" cause.

## Alertmanager routing review points

- `severity` labels must map to real routes (`page` → PagerDuty/phone, `ticket` → queue,
  `info` → nothing). An alert with a severity no route matches lands in the default receiver —
  verify where that actually goes.
- `group_by` too coarse (cluster-wide grouping) hides simultaneous distinct incidents; too fine
  (by pod) recreates the page storm that grouping exists to prevent. Service-level is the
  default answer.
- Inhibition: node-down should inhibit the per-service alerts on that node; check inhibit rules
  exist for the obvious cascade pairs (node → pods, upstream → downstream).
- `repeat_interval` < shift length means the same unresolved incident re-pages the same person;
  4h–12h for pages is typical.
