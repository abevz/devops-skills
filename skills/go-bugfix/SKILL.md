---
name: go-bugfix
description: Use when fixing a bug in Go code. Mention "fix this go bug", "this go function is broken", "go bugfix" as triggers.
license: MIT
compatibility: Works with Claude Code, Codex-style agents, CodeWhale, OpenCode, and other agents that support Agent Skills-style instructions.
---

# Go Bugfix

## When to use

Use when a Go program has a specific, identifiable bug (wrong output, panic, failing test,
deadlock) that needs a fix.

## Goal

Fix the actual bug with a minimal, correct change, backed by a regression test — without
refactoring unrelated code.

## Workflow

1. Reproduce or precisely understand the failing behavior: run the failing test, reproduce the
   panic, or get the exact wrong output from the user.
2. Inspect existing tests around the affected code to understand intended behavior and existing
   coverage.
3. Trace the code path to find the root cause (not just where the symptom surfaces — e.g. a nil
   pointer panic three calls downstream from where a value was never initialized).
4. Implement the minimal fix at the root cause.
5. Add a regression test that fails before the fix and passes after it.
6. Run the focused test(s) (`go test -run <TestName> ./...`) to confirm, then run the broader
   package tests to check for regressions.

## Safety rules

- Do not refactor unrelated code, rename things, or "clean up while you're in there" — fix only
  the bug.
- Do not silently swallow the error the bug caused (`recover()` without acting on it, ignoring
  an error return) — fix the cause, not the symptom.
- If the root cause is unclear after investigation, say so rather than applying a fix that only
  masks the symptom.

## Output format

```
## Bug
<what's wrong, how it manifests>

## Root cause
<file:line, why it happens>

## Fix
<the diff/change, minimal and scoped to the bug>

## Regression test
<the test added, and confirmation it failed before / passes after>

## Test run result
```

## Quality checklist

- [ ] The fix addresses the root cause, not just where the symptom appeared
- [ ] A regression test was added and shown to fail before the fix
- [ ] No unrelated code was refactored or renamed
- [ ] Focused tests were run and pass after the fix
- [ ] Errors are handled, not silently discarded
