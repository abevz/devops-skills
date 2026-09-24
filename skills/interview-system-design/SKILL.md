---
name: interview-system-design
description: Use when practicing or structuring an answer to a system design interview question. Mention "system design interview", "practice system design", "how would I design X in an interview" as triggers.
license: MIT
compatibility: Works with Claude Code, Codex-style agents, CodeWhale, OpenCode, and other agents that support Agent Skills-style instructions.
---

# Interview System Design

## TL;DR checklist

- [ ] Clarify functional and non-functional requirements.
- [ ] Estimate scale before choosing components.
- [ ] Explain the request path, trade-offs, and failure behavior.

## Key read-only checks

- Review the given prompt, assumptions, scale figures, and design constraints.

## Common pitfalls

- Do not start with a favorite technology before stating the need.

## Agent procedure

Follow the [Workflow](#workflow), [Safety rules](#safety-rules), and [Quality checklist](#quality-checklist) below.

## Quick references

- [Workflow](#workflow) below.

Last verified: unverified

## When to use

Use when preparing for a system design interview: practicing a specific question ("design a URL
shortener / rate limiter / notification system"), structuring an answer, or getting feedback on
a practice answer.

## Goal

Produce (or coach toward) a structured, senior-level system design answer that demonstrates the
things interviewers actually score: requirement clarification, reasoned tradeoffs, and knowing
where the design breaks — not memorized architecture diagrams.

## Workflow

1. **Clarify requirements first** — never start designing. Functional requirements (what must it
   do, for whom), then non-functional (scale, latency targets, consistency vs availability,
   durability). In practice mode, make the candidate ask these questions before revealing the
   answers.
2. **Estimate the scale** — back-of-envelope: users, requests/sec, data volume, read/write
   ratio. Derive it out loud (e.g. "10M DAU × 10 requests/day ≈ 1,200 rps average, plan for
   5–10× peak") — the reasoning is scored, not the number.
3. **High-level design** — name the major components (clients, API layer, services, data
   stores, queues, caches) and the request flow through them. Keep it simple enough to draw in
   two minutes; resist premature detail.
4. **Data model & API sketch** — key entities, the primary access patterns, and the API surface
   for the core flows. Choose storage per access pattern (relational vs KV vs blob vs search),
   stating why.
5. **Deep-dive the interesting parts** — pick the 1–2 components where this specific problem
   actually lives (the fan-out in a feed, the counter contention in a rate limiter, the ID
   generation in a shortener) and go deep there. Depth in the right place beats breadth.
6. **State the tradeoffs explicitly** — every choice gets its alternative and the reason:
   "cache-aside vs write-through here because...", "eventual consistency is acceptable for X
   but not for Y because...". This is the highest-signal section for the interviewer.
7. **Find the bottlenecks and failure modes** — what breaks first under 10× load, what happens
   when a component dies (single points of failure, retry storms, thundering herd), and how the
   design degrades.
8. **Close with evolution** — what you'd build for v1 versus what the design grows into, and
   what you'd monitor to know when to evolve it.

In **feedback mode** (reviewing a practice answer): score each of the eight steps present/weak/
missing, and give the 2–3 highest-impact improvements — not a full rewrite of the answer.

## Safety rules

- Do not skip requirement clarification and jump to architecture — that's the most common
  interview failure and this skill exists to prevent it.
- Do not present one "correct" design; present the tradeoff space and a justified choice. If a
  choice is genuinely contested, say so.
- Keep the level appropriate to a spoken interview: no code, no exhaustive configs — components,
  flows, numbers, and reasoning.
- In practice mode, don't give the answer away before the candidate has attempted each step.

## Output format

```
## Requirements (functional / non-functional)
## Scale estimate
<derived out loud>
## High-level design
<components + request flow>
## Data model & API
## Deep dive: <the 1-2 parts that matter for this problem>
## Tradeoffs
<choice → alternative → why>
## Bottlenecks & failure modes
## Evolution & monitoring
```

For feedback mode: per-step scoring (present / weak / missing) + top 2–3 improvements.

## Quality checklist

- [ ] Requirements were clarified before any design appeared
- [ ] Scale numbers are derived, not asserted
- [ ] The deep dive targets where this specific problem is hard
- [ ] Every major choice has a stated alternative and reason
- [ ] Failure modes and the first bottleneck are named
