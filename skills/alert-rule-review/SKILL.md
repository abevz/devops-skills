---
name: alert-rule-review
description: Use when writing or reviewing Prometheus alerting rules, PromQL expressions, or alert routing. Mention "review this alert rule", "promql alert", "why is this alert noisy" as triggers.
license: MIT
compatibility: Works with Claude Code, Codex-style agents, CodeWhale, OpenCode, and other agents that support Agent Skills-style instructions.
---

# Alert Rule Review

## TL;DR checklist

- [ ] Decide whether the rule is a page, warning, or ticket.
- [ ] Check PromQL windows, labels, counter handling, and absent-data behavior.
- [ ] Verify alert duration, routing, and runbook link.

## Key read-only checks

- Read the expression, scrape interval, labels, routing config, and related tests.

## Common pitfalls

- Do not page on a cause-only metric when the user-visible symptom is missing.

## Agent procedure

Follow the [Workflow](#workflow), [Safety rules](#safety-rules), and [Quality checklist](#quality-checklist) below.

## Quick references

- [references/promql-patterns.md](references/promql-patterns.md)

Last verified: unverified

## When to use

Use when writing new Prometheus alerting rules or reviewing existing ones — including "this
alert is too noisy" and "this incident had no alert" complaints. Complements
`observability-review` (which audits the whole signal landscape); this skill goes deep on
individual rules.

## Goal

Ensure every alert is symptom-based, actionable, correctly expressed in PromQL, and routed with
the right urgency — and that the rule set as a whole would have caught recent incidents.

## Workflow

1. **Symptom vs cause** — page-severity alerts fire on user-visible symptoms (error rate,
   latency, availability), not on causes (one pod restarted, CPU is high). Cause-based rules are
   fine as tickets/warnings, not pages.
2. **PromQL correctness** — check the expression itself:
   - rate/increase windows sized ≥ 4× scrape interval (a `rate(x[1m])` on 30s scrapes is noise)
   - aggregation labels: does `sum by (...)` keep the labels the notification needs and drop the
     ones that explode cardinality?
   - counter resets handled (`rate`/`increase`, never raw counter comparison)
   - division guarded against zero/absent denominators
3. **Absence blindness** — a rule on `rate(errors[5m]) > x` silently stops working when the
   target disappears entirely. Check for paired `absent()`/`up == 0` coverage on anything
   critical.
4. **`for:` duration** — long enough to survive blips (scrape hiccup, single slow request),
   short enough that the page still arrives while it matters. `for: 0` on a page needs
   justification.
5. **Severity & routing** — labels (`severity: page/ticket/info`) match real urgency and the
   routing config actually uses them; nothing user-critical routes to a channel nobody reads.
6. **Annotations** — `summary` states what's wrong for whom; `description` includes the
   offending label values (`{{ $labels.namespace }}`, `{{ $value }}`); `runbook_url` present and
   pointing at a real runbook (see `runbook-writer`).
7. **Noise audit** — for existing rules: which fired most in the last weeks without action taken?
   An alert that's always acknowledged-and-ignored is worse than none — recommend demoting,
   tuning, or deleting it explicitly.
8. **SLO burn-rate option** — for services with SLOs, prefer multi-window burn-rate alerts
   (fast-burn page + slow-burn ticket) over static thresholds, and say when that's overkill.
9. **Recording rules** — expensive expressions used by several alerts/dashboards belong in
   recording rules; check naming follows `level:metric:operations` convention.

## References

- `references/promql-patterns.md` — burn-rate math with the standard multiwindow table,
  absence-pairing patterns, rate/increase/histogram_quantile pitfalls, for:/keep_firing_for
  flap control, cardinality rules, recording-rule promotion criteria, and Alertmanager routing
  review points.

## Safety rules

- This is a review skill: do not apply rule changes to a live Prometheus/Alertmanager, and do
  not silence, inhibit, or delete active alerts.
- When recommending deletion of a noisy alert, state explicitly what coverage is lost and what
  (if anything) replaces it — never just "delete it."
- Do not invent SLO targets; take them from the user or mark as a decision they need to make.

## Output format

```
## Broken rules (wrong PromQL / will not fire when needed)
## Noisy or non-actionable rules
## Coverage gaps (incidents or failure modes with no alert)
## Rule improvements
<per rule: current → suggested expression/labels/annotations, with rationale>
## Routing issues
```

## Quality checklist

- [ ] Every page-severity rule was checked for symptom-vs-cause
- [ ] Absence/`up` coverage was checked, not just threshold rules
- [ ] rate() windows were checked against scrape interval
- [ ] Every recommended deletion states the coverage lost
- [ ] No live alerting config was modified or silenced
