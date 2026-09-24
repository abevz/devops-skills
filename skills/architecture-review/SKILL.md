---
name: architecture-review
description: Use when reviewing the architecture of a repository or system for boundaries, coupling, and long-term maintainability. Mention "architecture review", "is this well structured", "review our system design" as triggers.
license: MIT
compatibility: Works with Claude Code, Codex-style agents, CodeWhale, OpenCode, and other agents that support Agent Skills-style instructions.
---

# Architecture Review

## TL;DR checklist

- [ ] Map components and dependency direction.
- [ ] Identify unclear boundaries, coupling, and shared mutable state.
- [ ] Prioritize changes by operational and maintenance cost.

## Key read-only checks

- Read package/service boundaries, interfaces, deployment dependencies, and the proposed diff.

## Common pitfalls

- Do not infer architecture from the directory tree alone.

## Agent procedure

Follow the [Workflow](#workflow), [Safety rules](#safety-rules), and [Quality checklist](#quality-checklist) below.

## Quick references

- [Workflow](#workflow) below.

Last verified: unverified

## When to use

Use when reviewing a repository's or system's structure — module boundaries, service
decomposition, dependency graph — rather than a single diff.

## Goal

Identify structural risks (tight coupling, unclear boundaries, operational complexity) that will
slow the team down or cause outages as the system grows, with concrete evidence from the code.

## Workflow

1. Map the major components/services/packages and how they depend on each other.
2. Check **boundaries**: does each component have a clear, single responsibility, or has scope
   crept across boundaries over time?
3. Check **coupling**: are components coupled through well-defined interfaces/APIs, or through
   shared mutable state, direct DB access across services, or import cycles?
4. Check **layering**: does the dependency direction match the intended layering (e.g. domain
   logic doesn't depend on infrastructure details)?
5. Check **operational complexity**: how many moving parts does a change here require touching?
   How many services must be deployed together?
6. Check **future maintainability**: what happens if this component needs to be replaced,
   scaled independently, or owned by a different team?
7. If a migration is implied by the findings, sketch the migration path at a high level and
   point to the `migration-plan` skill for the detailed plan.
8. Optionally sketch the component map as a diagram (e.g. with `d2`, if the user has it
   installed) to make coupling/boundary issues easier to see — text form is always sufficient,
   the diagram is a bonus.

## Safety rules

- Do not propose a full rewrite as the default recommendation — prefer incremental,
  reversible steps.
- Ground every finding in actual code/config evidence (specific files, imports, or call paths),
  not general architectural opinions.
- Acknowledge tradeoffs — flag when a "smell" was a deliberate, reasonable tradeoff for the
  team's constraints (team size, timeline, scale).

## Output format

```
## Component map
<brief list of major components and their relationships>

## Boundary issues
## Coupling issues
## Layering issues
## Operational complexity concerns
## Migration path (if relevant)
<see migration-plan skill for a detailed rollout>

## What's working well
```

## Quality checklist

- [ ] Every finding cites specific code/config evidence, not just a general impression
- [ ] Deliberate, reasonable tradeoffs are acknowledged rather than flagged as bugs
- [ ] Recommendations favor incremental change over full rewrites
- [ ] Operational complexity (deploy/on-call impact) was explicitly considered
- [ ] "What's working well" is included, not just criticism
