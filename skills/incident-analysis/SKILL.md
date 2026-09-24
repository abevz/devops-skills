---
name: incident-analysis
description: Use when analyzing a production-like incident and writing it up. Mention "incident", "postmortem", "outage writeup" as triggers.
license: MIT
compatibility: Works with Claude Code, Codex-style agents, CodeWhale, OpenCode, and other agents that support Agent Skills-style instructions.
---

# Incident Analysis

## TL;DR checklist

- [ ] Collect the incident timeline and actual evidence.
- [ ] Separate user impact, immediate trigger, root cause, and mitigation.
- [ ] Write owned follow-up actions from the observed gaps.

## Key read-only checks

- Read alerts, logs, dashboards, deploy history, and recorded response actions.

## Common pitfalls

- Do not infer a root cause only from the first visible symptom.

## Agent procedure

Follow the [Workflow](#workflow), [Safety rules](#safety-rules), and [Quality checklist](#quality-checklist) below.

## Quick references

- [Workflow](#workflow) below.

Last verified: unverified

## When to use

Use after a production (or production-like) incident, when the user wants a structured writeup:
what happened, what broke, and what should change.

## Goal

Produce a clear, honest incident report that a team can act on — including what is still
uncertain.

## Workflow

1. Collect all available evidence: alerts, logs, dashboards, chat/incident-channel timeline,
   deploy history, on-call actions taken.
2. Establish the timeline in UTC (or the org's standard timezone), from first signal to
   resolution.
3. Determine blast radius: which users, services, regions, or data were affected, and for how
   long.
4. Separate the immediate trigger from the underlying root cause (they are often different).
5. Document what mitigated the incident (rollback, scale-up, feature flag, manual intervention)
   versus what will permanently fix it.
6. Extract concrete follow-up tasks, each with an owner placeholder and priority.
7. Capture lessons learned, including process or tooling gaps that slowed detection or response.

## Safety rules

- Do not hide or minimize uncertainty — mark unresolved questions as "unknown" or "unconfirmed"
  rather than filling gaps with plausible-sounding guesses.
- Do not assign blame to individuals; focus on systems and process.
- Do not mark the incident as fully resolved unless the permanent fix is verified, not just the
  symptom mitigated.

## Output format

```
## Summary
<1-2 sentences: what broke, for whom, for how long>

## Impact
<users/services/data affected, severity, duration>

## Timeline
<UTC timestamps, chronological>

## Root cause
<confirmed cause, distinct from the immediate trigger>

## Mitigation
<what stopped the bleeding, and when>

## Permanent fix
<what fully resolves the root cause, status>

## Follow-up tasks
- [ ] <task> — owner: TBD — priority: <high/med/low>

## Lessons learned
<detection gaps, response gaps, process changes>
```

## Quality checklist

- [ ] Timeline uses consistent timestamps and is in chronological order
- [ ] Root cause and immediate trigger are clearly distinguished
- [ ] Every follow-up task is concrete and actionable, not vague ("improve monitoring")
- [ ] Uncertainty is explicitly marked, not glossed over
- [ ] No individual is blamed; findings are systemic
