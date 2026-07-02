---
name: migration-plan
description: Use when creating a safe migration plan for infrastructure, a service, a data store, or a platform change. Mention "migration plan", "how do we migrate", "rollout plan" as triggers.
license: MIT
compatibility: Works with Claude Code, Codex-style agents, CodeWhale, OpenCode, and other agents that support Agent Skills-style instructions.
---

# Migration Plan

## When to use

Use when planning a move from one system, platform, or configuration to another — e.g. ingress
controller swaps, database engine changes, cloud provider moves, API version upgrades, or
cluster migrations — before any execution begins.

## Goal

Produce a phased, reversible migration plan that a team can execute safely, with explicit
validation and rollback at every phase.

## Workflow

1. **Current state** — describe what exists today precisely enough that "done" is unambiguous.
2. **Target state** — describe the desired end state and why it's needed.
3. **Risks** — list what could go wrong at each phase (data loss, downtime, traffic loss,
   split-brain, cost spike) and its likelihood/impact.
4. **Phased rollout** — break the migration into small, independently-verifiable phases. Prefer
   running old and new in parallel (dual-write, canary traffic, shadow deployment) over a
   big-bang cutover wherever feasible.
5. **Rollback plan** — for each phase, define exactly how to revert, and how long that stays
   possible before the point of no return.
6. **Validation plan** — define what "this phase succeeded" means concretely (metrics, smoke
   tests, error budgets) before moving to the next phase.
7. **Communication notes** — who needs to be informed, when, and what a maintenance-window/status
   message should say.

## Safety rules

- Do not execute any migration step yourself (no `kubectl apply`, `terraform apply`, DB
  migrations, or DNS changes) — this skill produces a plan for humans/CI to execute.
- Every phase must have a rollback path; if one doesn't exist, say so explicitly as a blocking
  risk rather than omitting it.
- Prefer reversible, incremental phases over irreversible big-bang cutovers.

## Output format

```
## Current state
## Target state
## Risks
<risk — likelihood — impact>

## Phased rollout
### Phase 1: <name>
- Actions:
- Validation:
- Rollback:
### Phase 2: ...

## Communication notes
```

## Quality checklist

- [ ] Every phase has an explicit rollback path
- [ ] Every phase has a concrete validation step, not just "check it works"
- [ ] The point of no return (if any) is called out explicitly
- [ ] Big-bang cutover was avoided unless genuinely unavoidable, and that's stated
- [ ] No migration step was executed by the agent
