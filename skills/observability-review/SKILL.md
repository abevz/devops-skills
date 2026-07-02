---
name: observability-review
description: Use when reviewing logs, metrics, traces, dashboards, alerts, or SLO readiness for a service. Mention "observability review", "are our alerts good", "SLO check" as triggers.
license: MIT
compatibility: Works with Claude Code, Codex-style agents, CodeWhale, OpenCode, and other agents that support Agent Skills-style instructions.
---

# Observability Review

## When to use

Use when reviewing whether a service can be understood and debugged in production: its logs,
metrics, traces, dashboards, alerts, and SLOs.

## Goal

Find the gaps between "we have telemetry" and "we can actually diagnose an incident with it in
under five minutes."

## Workflow

1. **Logs** — check for structured (not free-text) logging, consistent correlation/request IDs,
   appropriate levels, and that errors include enough context to act on (not just "request
   failed").
2. **Metrics** — check for RED (Rate, Errors, Duration) on request-driven services and USE
   (Utilization, Saturation, Errors) on resources; flag missing error-rate or latency-percentile
   metrics as high priority.
3. **Traces** — check whether cross-service calls are traceable end-to-end, and whether trace IDs
   are correlated with logs.
4. **Dashboards** — check whether a dashboard exists that answers "is this service healthy right
   now" at a glance, versus one that's just a wall of raw graphs.
5. **Alerts** — check each alert for: does it fire on a symptom that matters to users, does it
   have a sane threshold, is it likely to be noisy (flapping, no hysteresis), and does it link to
   a runbook.
6. **Runbooks** — check that alerts and dashboards link to a runbook or troubleshooting guide, not
   just a raw query.
7. **Tracing/metric gaps** — call out the specific missing signal that would have shortened the
   last incident, if known.

## References

- `references/instrumentation.md` — probes as the observability foundation, Prometheus scraping
  (annotations vs ServiceMonitor), a hardened instrumented-deployment checklist, metric
  naming/buckets, alert-vs-dashboard guidance, and logging/tracing patterns for Kubernetes.

## Safety rules

- This is a review skill: do not modify alerting rules, dashboards, or monitoring config directly
  unless explicitly asked.
- Flag noisy or non-actionable alerts explicitly — silence on this is a common failure mode.
- Do not assume telemetry exists just because the code "should" emit it; verify from actual
  dashboards/queries/config where possible, and mark anything unverified.

## Output format

```
## Useful signals already present
## Logging gaps
## Metrics gaps (RED/USE)
## Tracing gaps
## Dashboard gaps
## Alert quality issues (noisy / non-actionable / missing runbook link)
## Top 3 improvements to make first
```

## Quality checklist

- [ ] Every alert reviewed was checked for noise/actionability, not just existence
- [ ] RED/USE coverage was checked explicitly, not assumed
- [ ] Missing runbook links are called out per alert, not just in general
- [ ] Recommendations are prioritized, not a flat list
- [ ] No monitoring config was changed without being asked
