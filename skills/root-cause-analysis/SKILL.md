---
name: root-cause-analysis
description: Use when investigating a bug, failure, regression, outage, or unclear behavior and the cause is not yet known. Mention "root cause", "why is this happening", "investigate this bug" as triggers.
license: MIT
compatibility: Works with Claude Code, Codex-style agents, CodeWhale, OpenCode, and other agents that support Agent Skills-style instructions.
---

# Root Cause Analysis

## When to use

Use when something is broken, behaving unexpectedly, or regressed, and the cause is not yet
established — before jumping to a fix.

## Goal

Find the actual root cause, backed by evidence, and propose a fix and a way to prevent
recurrence — without guessing.

## Workflow

1. **Symptoms** — write down exactly what is observed (error message, wrong output, crash,
   latency spike) and what was expected instead.
2. **Timeline** — when did it start? What changed around that time (deploy, config, dependency
   bump, traffic pattern, data shape)?
3. **Evidence** — gather logs, stack traces, metrics, recent commits/diffs, test failures.
   Quote exact evidence; do not paraphrase from memory.
4. **Hypotheses** — list plausible causes ranked by likelihood, each tied to specific evidence
   that would confirm or rule it out.
5. **Validation** — test each hypothesis (reproduce locally, add a targeted log/print, bisect the
   commit history, read the exact code path) until one is confirmed.
6. **Root cause** — state the confirmed cause in one or two sentences, distinct from any
   contributing/secondary factors.
7. **Fix** — propose the minimal correct fix for the root cause, not just the symptom.
8. **Prevention** — suggest how to prevent this class of bug (test, assertion, lint rule,
   monitoring, process change).

## Safety rules

- Do not state a cause as confirmed until it has been validated with evidence.
- Clearly separate "evidence" (what was observed) from "assumption" (what you believe but haven't
  checked) at every step.
- If two hypotheses remain equally plausible after investigation, say so rather than picking one
  arbitrarily.
- Do not apply the fix automatically unless asked — propose it first.

## Output format

```
## Symptoms
## Timeline
## Evidence
## Hypotheses (ranked)
## Validation
## Root cause
## Fix
## Prevention
```

## Quality checklist

- [ ] Every claim in "Root cause" is backed by evidence listed above it
- [ ] Assumptions are explicitly labeled as assumptions, not stated as fact
- [ ] At least one alternative hypothesis was considered and ruled out
- [ ] The fix addresses the root cause, not just the symptom
- [ ] A prevention measure is proposed, not just the fix
