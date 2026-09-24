---
name: grafana-dashboards
description: Use when designing, building, or reviewing Grafana dashboards. Mention "grafana dashboard", "build a dashboard", "review this dashboard" as triggers.
license: MIT
compatibility: Works with Claude Code, Codex-style agents, CodeWhale, OpenCode, and other agents that support Agent Skills-style instructions.
---

# Grafana Dashboards

## TL;DR checklist

- [ ] Name the dashboard audience and the question it must answer.
- [ ] Put the service or resource verdict before diagnostic detail.
- [ ] Check panel queries, units, variables, and drill-down links.

## Key read-only checks

- Read dashboard JSON or panel queries plus the metrics they use.

## Common pitfalls

- Do not make a dashboard dense enough to hide the headline signal.

## Agent procedure

Follow the [Workflow](#workflow), [Safety rules](#safety-rules), and [Quality checklist](#quality-checklist) below.

## Quick references

- [references/panel-patterns.md](references/panel-patterns.md)

Last verified: unverified

## When to use

Use when creating a new Grafana dashboard, restructuring an unusable one, or reviewing dashboard
JSON/queries — for services, clusters, or SLOs.

## Goal

Produce a dashboard that answers a specific question for a specific audience in seconds — not a
wall of every metric that exists.

## Workflow

1. **Audience and question first** — before any panel: who looks at this (on-call, service
   owner, capacity planning) and what question must it answer at a glance ("is the service
   healthy right now?" is a different dashboard from "why was it slow yesterday?"). One
   dashboard, one audience.
2. **Top row = verdict** — the first row answers the headline question: RED metrics (rate,
   errors, duration) for request-driven services, USE (utilization, saturation, errors) for
   resources, or SLO status/burn rate. Stat panels with thresholds, readable from across the
   room.
3. **Drill-down structure** — rows below decompose the verdict: per-endpoint latency, per-pod
   errors, dependency health. Collapsed rows for detail nobody needs by default. Link out to
   related dashboards (node-level, dependency services) instead of duplicating their panels.
4. **Template variables** — `$namespace`, `$service`, `$pod` etc. from `label_values()` queries,
   with sane defaults and `All` only where the queries survive it. One parameterized dashboard
   beats N copied ones — copies drift.
5. **Query hygiene** — same rules as alert PromQL: rate windows sized to scrape interval (or use
   `$__rate_interval`), recording rules for expensive queries reused across panels, no
   high-cardinality group-bys that render 500 series into one panel.
6. **Readability** — units set on every panel (seconds vs ms bites everyone), consistent color
   semantics (red = bad everywhere), legends with meaningful labels (`{{pod}}`, not the raw
   expression), percentiles labeled (p50/p95/p99, never just "latency"), Y-axes from zero unless
   there's a stated reason.
7. **Thresholds and annotations** — panel thresholds mirror the actual alert thresholds (a
   green panel while the alert fires destroys trust); deploy/incident annotations wired in so
   "what changed at 14:32" is visible on every graph.
8. **Dashboards as code** — dashboard JSON (or Jsonnet/grafonnet, or a Grafana provisioning
   ConfigMap/CRD) lives in git and deploys via provisioning/GitOps, not hand-edited in the UI.
   Flag UI-only dashboards as drift risk; `uid` pinned so links survive re-provisioning.
9. **Review pass** (for existing dashboards) — panels nobody can explain, broken queries
   (`No data` for months), duplicated panels that contradict each other, and missing top-row
   verdict are the four most common findings.

## References

- `references/panel-patterns.md` — verdict-panel query patterns (with `$__rate_interval`),
  chained template variables, legend/unit/axis discipline, deploy annotations, provisioning
  shapes compared (sidecar ConfigMap / operator CRD / file / Terraform), uid pinning, and
  review quick-hits for existing dashboards.

## Safety rules

- Do not modify or delete live Grafana dashboards, datasources, or provisioning config unless
  explicitly asked — produce the JSON/queries for the user to review and apply.
- Do not invent metric names — verify they exist in the user's stack (from code, /metrics
  output, or the user), or mark them `TODO(verify metric name)`.
- When reviewing, don't recommend deleting panels without stating what visibility is lost.

## Output format

For a new dashboard:
```
## Audience & headline question
## Layout
<row-by-row: panels, type, query, unit, thresholds>
## Template variables
## Dashboard JSON (or grafonnet)
## Provisioning notes
<how this should land in git/GitOps rather than the UI>
```

For a review: findings grouped as **Broken / Misleading / Unreadable / Structure**, each with a
concrete fix.

## Quality checklist

- [ ] Top row answers the headline question without scrolling
- [ ] Every panel has correct units and labeled percentiles where relevant
- [ ] Panel thresholds match the corresponding alert thresholds
- [ ] All metric names are verified or marked TODO
- [ ] The dashboard is deliverable as code (JSON in git), not a UI-only artifact
