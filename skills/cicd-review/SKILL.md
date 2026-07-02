---
name: cicd-review
description: Use when reviewing a CI/CD pipeline definition such as GitHub Actions workflows or GitLab CI config for security and reliability. Mention "review this workflow", "ci review", "is this pipeline safe" as triggers.
license: MIT
compatibility: Works with Claude Code, Codex-style agents, CodeWhale, OpenCode, and other agents that support Agent Skills-style instructions.
---

# CI/CD Review

## When to use

Use when reviewing pipeline definitions — GitHub Actions workflows, GitLab CI, or similar —
for security holes, supply-chain risk, and reliability problems. This reviews the *pipeline
itself*, not the application code it builds.

## Goal

Find the ways this pipeline can be abused (secret exfiltration, privilege escalation, supply
chain) or can silently misbehave (flaky, unpinned, unbounded), ranked by severity.

## Workflow

1. **Permissions** — GitHub Actions: top-level `permissions:` set to least privilege (ideally
   `contents: read` default, job-level escalation only where needed); flag workflows running
   with the default broad token.
2. **Untrusted input injection** — any `${{ github.event.* }}` (PR title, branch name, issue
   body, commit message) interpolated directly into `run:` is command injection from anyone who
   can open a PR. Fix: pass through `env:` and quote, or avoid entirely.
3. **`pull_request_target` / privileged triggers** — flag any workflow combining a privileged
   trigger (secrets access, write token) with a checkout of untrusted PR code. This is the
   classic secret-exfiltration pattern.
4. **Action/image pinning** — third-party actions pinned to a full commit SHA (not a mutable
   tag like `@v3` for anything with secrets access); container images used in jobs pinned by
   digest or exact version.
5. **Secrets handling** — secrets not echoed into logs (watch `set -x`, debug steps, `env`
   dumps); scoped to the jobs that need them (environments/protected branches); OIDC/workload
   identity preferred over long-lived cloud keys stored as secrets.
6. **Deploy gates** — anything that applies to production (`kubectl apply`, `terraform apply`,
   `helm upgrade`, deploy steps) sits behind a review gate: protected environment, required
   approval, or at minimum a separate manually-triggered workflow — never on every push to a
   branch anyone can push to.
7. **Reliability** — `timeout-minutes` on jobs (default is 6 hours of burned runners);
   `concurrency` groups to cancel superseded runs on the same ref; caches keyed correctly
   (lockfile hash, not a static key that never invalidates).
8. **Cache/artifact poisoning** — caches writable from PR branches and readable by privileged
   workflows are an escalation path; artifacts passed between workflows validated before use.
9. **Reproducibility** — build steps don't `curl | sh` unpinned installers; tool versions pinned
   (setup actions with explicit versions, or versions from the repo's own config files).

## Safety rules

- This is a review skill: do not edit workflows, trigger runs, re-run jobs, or modify repository/
  CI settings.
- Never echo a secret value found in logs or config — describe the exposure without quoting it.
- Treat "untrusted input reaches a privileged context" findings as Critical even if exploitation
  looks unlikely today — the pipeline outlives the current threat model.

## Output format

```
## Critical (exploitable: injection, secret exposure, privileged untrusted code)
<finding — attack path — fix>

## Major (supply chain / unsafe deploy path)
## Reliability
## Minor / style
```

## Quality checklist

- [ ] Every `${{ }}` interpolation into a shell step was checked for untrusted input
- [ ] `pull_request_target` (and equivalents) usage was checked against untrusted checkouts
- [ ] Third-party actions were checked for SHA pinning where they see secrets
- [ ] Production deploy steps were checked for a review gate
- [ ] No workflow was modified or triggered
