---
name: runbook-writer
description: Use when writing or improving an operational runbook for an alert, service, or recurring failure. Mention "write a runbook", "runbook for this alert", "on-call doc" as triggers.
license: MIT
compatibility: Works with Claude Code, Codex-style agents, CodeWhale, OpenCode, and other agents that support Agent Skills-style instructions.
---

# Runbook Writer

## TL;DR checklist

- [ ] State the triggering alert and user impact.
- [ ] Put ordered read-only diagnosis before action.
- [ ] Map each mitigation to the evidence that justifies it.

## Key read-only checks

- Read the alert rule, dashboards, logs, and existing recovery instructions.

## Common pitfalls

- Do not offer an invasive command without a condition and a recovery path.

## Agent procedure

Follow the [Workflow](#workflow), [Safety rules](#safety-rules), and [Quality checklist](#quality-checklist) below.

## Quick references

- [Workflow](#workflow) below.

Last verified: unverified

## When to use

Use when an alert, service, or recurring incident needs a runbook — or when an existing runbook
failed the on-call engineer during the last incident and needs fixing.

## Goal

Produce a runbook a half-awake on-call engineer who has never touched this service can follow at
3 AM: what fired, what it means for users, exactly what to check, and exactly what to do —
copy-pasteable, in order, with destructive steps unmistakably marked.

## Workflow

1. Identify the trigger: which alert(s) or symptom this runbook covers, and what the alert
   condition actually means in plain language.
2. State user impact honestly: what is broken for whom while this fires, and how urgent it
   really is (drives whether the reader pages someone else or has coffee first).
3. Write the diagnosis sequence: ordered, copy-pasteable, read-only commands (kubectl/logs/
   PromQL/dashboard links), each with one line saying what output means what. Order by
   likelihood × cheapness — most likely cause checked first.
4. Write mitigations mapped to diagnoses: "if you saw X in step 3, do Y." Start with the safest
   effective action (scale up, feature-flag off, shift traffic) before invasive ones.
5. Mark every mutating command with an explicit warning line stating what it changes and when
   NOT to run it. Rollbacks/restarts/deletions never appear bare.
6. Define escalation: when to stop (time-boxed — e.g. "15 minutes without progress"), who to
   page (team/role, not a person's name), and what context to hand over.
7. Define verification: how to confirm recovery (the metric/query that should return to normal)
   — not just "the alert stopped firing."
8. Link related material: dashboard, recent incidents of this class, architecture doc, the
   alert rule definition itself.

## Safety rules

- Do not run any of the commands while writing the runbook — this skill produces a document, not
  an intervention. Verify command *syntax* against the manifests/config available, and mark
  anything unverifiable as `# UNVERIFIED — check before relying on this`.
- Every mutating command must carry an explicit warning and a "when not to do this" note.
- Do not invent thresholds, dashboard URLs, or team names — take them from provided context or
  leave a clearly marked `TODO(owner)` placeholder.

## Output format

```
# Runbook: <alert or failure name>

## Alert / trigger
## User impact & urgency
## Diagnosis (in order)
1. <command> — <what result means>
## Mitigation
- If <diagnosis result>: <action>   ⚠️ marked if mutating
## Escalation
## Verify recovery
## Links
```

## Quality checklist

- [ ] Every diagnosis command is copy-pasteable and read-only
- [ ] Every mutating command has a warning and a "when not to" note
- [ ] Escalation is time-boxed with a concrete page target
- [ ] Recovery verification is a concrete signal, not "alert resolved"
- [ ] Unknown values are marked TODO, not invented
