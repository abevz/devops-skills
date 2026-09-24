---
name: go-testing-review
description: Use when reviewing or improving Go tests for coverage, quality, or reliability. Mention "review these go tests", "improve test coverage", "go test review" as triggers.
license: MIT
compatibility: Works with Claude Code, Codex-style agents, CodeWhale, OpenCode, and other agents that support Agent Skills-style instructions.
---

# Go Testing Review

## TL;DR checklist

- [ ] Check behavior coverage across success, error, boundary, and concurrency cases.
- [ ] Look for deterministic tests and narrow fakes.
- [ ] Identify missing tests that would catch a real regression.

## Key read-only checks

- Read production behavior and test assertions; inspect race-test output when supplied.

## Common pitfalls

- Do not reward coverage numbers when assertions miss the failure mode.

## Agent procedure

Follow the [Workflow](#workflow), [Safety rules](#safety-rules), and [Quality checklist](#quality-checklist) below.

## Quick references

- [Workflow](#workflow) below.

Last verified: unverified

## When to use

Use when reviewing existing Go tests, or deciding what tests to add, for a package or change.

## Goal

Find real gaps in test quality and coverage — missing edge cases, non-deterministic tests, weak
assertions — not just raw coverage percentage.

## Workflow

1. Check **table-driven tests** are used where multiple similar cases exist, instead of repeated
   near-identical test functions.
2. Check **edge cases**: empty input, nil, zero values, max/min boundaries, duplicate entries —
   are these covered, or only the happy path?
3. Check **error paths**: does every function that returns an error have a test that triggers
   that error and asserts on it (not just the success case)?
4. Check **concurrency cases**: if the code under test uses goroutines/channels, are there tests
   that exercise concurrent access, and is `go test -race` clean?
5. Check **mocks/fakes**: are they narrow interfaces matching real usage, or brittle mocks that
   assert on implementation details rather than behavior?
6. Check **determinism**: no reliance on real time, real network, unseeded randomness, or test
   execution order — flag anything that could flake in CI.
7. Check **test readability**: descriptive test/subtest names (`t.Run("empty input returns
   ErrInvalid", ...)`), clear arrange/act/assert structure, no unexplained magic values.
8. Run the tests (`go test ./... -race`) if possible, and report actual results, not assumed
   ones.

## Safety rules

- This is a review skill: do not delete or drastically rewrite existing tests without asking,
  even if they look redundant — confirm intent first.
- Do not claim a test "covers" a case unless the assertion would actually fail if that behavior
  broke — flag tests that run code but assert nothing meaningful.
- Do not recommend chasing 100% coverage over testing behavior that actually matters.

## Output format

```
## Missing coverage (edge cases / error paths not tested)
## Flaky / non-deterministic risks
## Weak assertions (tests that wouldn't catch a real regression)
## Readability issues
## Suggested new tests
<table-driven skeletons where applicable>
## Test run result
```

## Quality checklist

- [ ] Error paths were checked, not just success paths
- [ ] Concurrency-related code was checked with `-race` if applicable
- [ ] Assertions were checked for whether they'd actually catch a regression
- [ ] No existing test was deleted/rewritten without confirming intent
- [ ] Actual test run output is reported when tests were run
