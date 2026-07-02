# Contributing / Maintenance Guide

Notes for adding, changing, or retiring skills in this repository later.

## Adding a new skill

1. **Check for overlap first.** Read the skill table in `README.md` and skim any skill that
   sounds close to what you want to add. Prefer extending an existing skill's workflow over
   creating a near-duplicate. If two skills would answer "use when reviewing Kubernetes YAML"
   almost identically, that's a sign to merge, not add a third.
2. **Create a directory**: `skills/<skill-name>/`, lowercase, hyphen-separated, matching the
   `name:` field exactly. No leading/trailing hyphen, no underscores, no spaces.
3. **Write `SKILL.md`** using the required frontmatter and section shape (see "Skill shape"
   below). Copy an existing skill as a starting template — `terraform-review` or
   `kubernetes-security` are good examples of the intended depth and tone.
4. **Keep it small.** A skill should cover one job (debug X, review Y, write Z), not a whole
   domain. If the workflow section is growing past what a human could realistically hold in
   their head while reading it once, it's probably two skills.
5. **Add `references/` only if it earns its place.** A `references/tools.md` listing optional
   CLI tools, or a longer checklist that would bloat `SKILL.md`, is a good reason. A single
   paragraph of extra context is not — put it inline instead.
6. **No `scripts/` unless truly unavoidable.** This repository is deliberately markdown-only
   (see `docs/design-notes.md` and `SECURITY.md`). If you're tempted to add a script, first ask
   whether the skill can just tell the agent which read-only command to run instead.
7. **Update the skill table in `README.md`** with the new skill's name, category, and one-line
   purpose.

## Skill shape (required)

```
---
name: skill-name
description: Use when <trigger>. Mention <keywords> as triggers.
license: MIT
compatibility: Works with Claude Code, Codex-style agents, CodeWhale, OpenCode, and other agents that support Agent Skills-style instructions.
---

# Skill Title

## When to use
## Goal
## Workflow
## Safety rules
## Output format
## Quality checklist
```

Frontmatter rules (matches the [Agent Skills specification](https://agentskills.io)):

- `name` — lowercase, hyphen-separated, ≤64 characters, must exactly match the directory name.
- `description` — starts with "Use when...", states the concrete trigger, ≤1024 characters.
- `license` — `MIT` for everything in this repository, for consistency.
- `compatibility` — keep the standard line above unless a skill genuinely only works on one
  platform.

## Removing or merging a skill

- If a skill turns out to overlap heavily with another, merge the better parts into one and
  delete the other — don't keep both "for reference." Update `README.md` and
  `docs/design-notes.md` to reflect the merge and why.
- If a skill is no longer useful, delete the directory and remove it from the README table.
  Don't leave stale entries.

## Importing ideas from third-party skill repositories

If you find another skill repository worth mining for ideas later, follow the same process used
to build this repository:

1. Clone it somewhere **outside** this repo (or into a path this repo's `.gitignore` excludes)
   — never as a tracked subdirectory here.
2. Read it fully before reusing anything; run the `grep` checklist in `SECURITY.md`.
3. Extract *ideas and wording*, not scripts, hooks, or install logic — rewrite content in this
   repository's own skill shape rather than copying files wholesale.
4. Add an entry to `docs/third-party-review.md` documenting what you found, what you reused, and
   the trust level you assigned it, the same way the initial batch was documented.
5. Never add a `postinstall` script, install hook, or auto-executing shell script as part of the
   import.

## Validating before committing

CI (`.github/workflows/validate.yml`) automatically checks structure on every push/PR:
frontmatter presence and required fields, `name` ↔ directory match, naming convention, the six
required sections, no executable scripts/package manifests, and no secret patterns. It validates
*shape*, not content quality — before committing, also manually check:

- [ ] `description` states a concrete, realistic trigger someone would actually type
- [ ] Safety rules cover the destructive commands this skill's domain could reach
- [ ] No install instructions were added (CI catches scripts, not prose telling an agent to install things)
- [ ] README skill table is updated
- [ ] For skills with meaningful failure modes: a RED/GREEN scenario pair added to
      `tests/baseline-scenarios.md` + `tests/compliance-verification.md`
