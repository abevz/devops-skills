---
name: git-message
description: Use when the agent needs to write a Git commit message from staged or unstaged changes. Mention "commit message", "write a commit", or "conventional commit" as triggers.
license: MIT
compatibility: Works with Claude Code, Codex-style agents, CodeWhale, OpenCode, and other agents that support Agent Skills-style instructions.
---

# Git Message

## When to use

Use when the user asks for a commit message, wants help writing one from a diff, or asks to
follow Conventional Commits for a change that is staged, unstaged, or described in words.

## Goal

Produce a high-quality, Conventional-Commits-style commit message that explains *why* the change
was made, not just what changed.

## Workflow

1. Inspect the diff (`git diff --staged`, falling back to `git diff` if nothing is staged).
2. Identify the change type: `feat`, `fix`, `refactor`, `docs`, `test`, `chore`, `perf`, `ci`,
   `build`, `style`.
3. Identify the scope (package, module, or service touched), if it adds clarity.
4. Draft a subject line: `<type>(<scope>): <imperative summary>`, ≤72 characters.
5. Draft a body: explain motivation and context, not a line-by-line restatement of the diff.
6. Check for breaking changes; if present, add a `BREAKING CHANGE:` footer describing the impact
   and migration path.
7. Note any risk (e.g. touches shared config, no tests added, affects a hot path) as a short
   "Risk notes" line.
8. Suggest the exact `git commit` command the user can run, but do not run it yourself.

## Safety rules

- Do not run `git commit`, `git push`, or stage files automatically unless the user explicitly
  asks you to.
- Do not invent details not visible in the diff or supplied by the user — if intent is unclear,
  say so instead of guessing.
- Never include secrets, tokens, or credentials that may appear in a diff; flag them instead of
  quoting them.

## Output format

```
Subject: <type>(<scope>): <summary>

Body:
<why this change was made, context, tradeoffs>

Breaking changes: <none | description + migration notes>

Risk notes: <none | short risk callout>

Suggested command:
git commit -m "<subject>" -m "<body>"
```

## Quality checklist

- [ ] Subject uses imperative mood and a valid Conventional Commits type
- [ ] Subject is ≤72 characters
- [ ] Body explains motivation, not just mechanics
- [ ] Breaking changes are called out explicitly if present
- [ ] No secrets or credentials are echoed from the diff
- [ ] Commit was not executed automatically
