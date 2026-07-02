---
name: go-code-review
description: Use when reviewing Go code for idiomatic style, correctness, and maintainability. Mention "review this go code", "is this idiomatic go", "go code review" as triggers.
license: MIT
compatibility: Works with Claude Code, Codex-style agents, CodeWhale, OpenCode, and other agents that support Agent Skills-style instructions.
---

# Go Code Review

## When to use

Use when reviewing Go code for quality — separate from `pr-review` (broader, any language) and
`go-testing-review` (tests specifically). Use this one when the focus is Go-specific code
quality itself.

## Goal

Catch non-idiomatic patterns, concurrency hazards, and maintainability issues specific to Go,
with concrete fixes.

## Workflow

1. Check **idiomatic Go**: naming (short, clear, no Hungarian notation), package structure
   (no `utils`/`common` dumping grounds), effective use of the standard library over
   reinventing it.
2. Check **context usage**: `context.Context` passed as the first parameter, not stored in a
   struct; cancellation/timeouts respected in long-running or I/O operations; no
   `context.Background()` used deep in call chains where a real context should propagate.
3. Check **error handling**: errors wrapped with `%w` for context, not swallowed or logged-and-
   ignored; sentinel errors compared with `errors.Is`, not `==`; no `panic` for ordinary error
   conditions.
4. Check **naming**: exported identifiers documented, receiver names short and consistent,
   interfaces named for behavior (`-er` convention) where idiomatic.
5. Check **package boundaries**: no import cycles, internal packages used to hide
   implementation details that shouldn't be part of the public API.
6. Check **concurrency safety**: shared state protected by a mutex or owned by a single
   goroutine; goroutines have a clear termination condition (no leaks); channels have a clear
   owner/closer.
7. Check **resource cleanup**: `defer Close()`/`defer cancel()` used correctly, not inside a
   loop in a way that delays cleanup until the function returns.
8. Check **logging**: structured, leveled logging instead of ad hoc `fmt.Println`/`log.Println`
   in library code.
9. Check **performance**: obvious unnecessary allocations in hot paths (e.g. repeated string
   concatenation instead of `strings.Builder`), but don't over-optimize code that isn't hot.
10. Check **security**: unchecked input passed to `exec.Command`, SQL built via string
    concatenation, unsafe deserialization, secrets logged or hardcoded.

## Safety rules

- Do not rewrite the whole file — point to specific lines and specific fixes.
- Distinguish idiom violations (style) from actual bugs (concurrency hazard, resource leak,
  security issue) in severity.
- Do not flag goroutine/channel patterns as leaks without tracing whether they actually have no
  termination condition — false positives here are common and costly to chase.

## Output format

```
## Correctness / concurrency issues
## Security issues
## Idiom / style issues
## Performance notes (only if in a demonstrably hot path)
## Suggestions
```

## Quality checklist

- [ ] Concurrency safety was traced (goroutine termination, shared state protection), not assumed
- [ ] Context propagation was checked end-to-end for I/O paths
- [ ] Errors are wrapped/compared idiomatically (`%w`, `errors.Is`/`As`)
- [ ] Findings are ranked by real impact (bug > idiom), not treated as equally severe
- [ ] No whole-file rewrite was proposed
