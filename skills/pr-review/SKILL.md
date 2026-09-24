---
name: pr-review
description: Use when the agent needs to review a pull request or diff as a lead engineer. Mention "review this PR", "code review", or "review my diff" as triggers.
license: MIT
compatibility: Works with Claude Code, Codex-style agents, CodeWhale, OpenCode, and other agents that support Agent Skills-style instructions.
---

# PR Review

## TL;DR checklist

- [ ] Identify the change intent and read the complete diff with callers.
- [ ] Check correctness, security, compatibility, performance, and meaningful tests.
- [ ] Report only actionable findings with evidence and severity.

## Key read-only checks

- Inspect the PR diff, affected call sites, and focused tests before judging.

## Common pitfalls

- Do not approve a change based only on a green CI badge.

## Agent procedure

Follow the [Workflow](#workflow), [Safety rules](#safety-rules), and [Quality checklist](#quality-checklist) below.

## Quick references

- [Workflow](#workflow) below.

Last verified: unverified

## When to use

Use when the user shares a pull request, diff, or branch and asks for a review, or wants a second
opinion before merging.

## Goal

Give an actionable, lead-engineer-level review that catches real defects and maintainability
risks without rewriting the whole change.

## Workflow

1. Understand the intent of the change (PR description, linked issue, or ask the user if unclear).
2. Read the full diff, not just the hunks — check callers and tests affected by each change.
3. Evaluate against these dimensions:
   - Correctness (logic errors, edge cases, off-by-one, nil/None handling, race conditions)
   - Maintainability (naming, structure, duplication, hidden coupling)
   - Readability (can a new engineer follow this in six months?)
   - Performance (obvious hot-path regressions, N+1 patterns, unnecessary allocations)
   - Security (injection, secrets in code, unsafe deserialization, auth/authz gaps)
   - Tests (are the new/changed paths covered? are the tests meaningful, not just coverage padding?)
   - Backward compatibility (API/schema/config changes that break existing callers)
4. Classify every finding by severity before writing prose.
5. For each finding, give the file/line, the concrete failure scenario, and a suggested fix.

## Safety rules

- Do not rewrite the whole PR or produce a full alternative implementation unless asked.
- Do not approve or merge anything — only the user or their CI/CD does that.
- Distinguish clearly between "this is broken" and "this is a style preference."
- If you are not confident a finding is real, say so instead of stating it as fact.

## Output format

```
## Critical
- <file:line> — <issue> — <why it breaks, concrete scenario> — <suggested fix>

## Major
- ...

## Minor
- ...

## Suggestions
- ...

## Summary
<1-3 sentences: is this safe to merge as-is, with fixes, or does it need rework?>
```
Omit any section with no findings rather than writing "None."

## Quality checklist

- [ ] Every Critical/Major finding has a concrete failure scenario, not just a vague concern
- [ ] Tests were checked, not assumed
- [ ] Security and backward-compatibility were explicitly considered
- [ ] No full rewrite was proposed unless requested
- [ ] Findings are ranked most-severe first
