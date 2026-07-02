---
name: terraform-review
description: Use when writing, reviewing, or debugging Terraform or OpenTofu code, modules, or plan output. Mention "terraform review", "review this module", "terraform plan looks wrong" as triggers.
license: MIT
compatibility: Works with Claude Code, Codex-style agents, CodeWhale, OpenCode, and other agents that support Agent Skills-style instructions.
---

# Terraform Review

## When to use

Use when reviewing Terraform/OpenTofu modules, variables/outputs, provider configuration, or
`plan` output — before anything is applied.

## Goal

Diagnose the actual failure mode or risk category present, and give a response with explicit
assumptions, risk, and a rollback-aware fix — not a generic "looks fine" or a wall of unranked
opinions.

## Workflow

1. Identify the failure mode category present, if any: **identity churn** (resource replacement
   forced by a rename/tag/provider change), **secrets** (hardcoded credentials, secrets in state
   or outputs), **blast radius** (a change that affects far more than intended, e.g. a
   module-wide variable change or a `count`/`for_each` key change reshuffling resource addresses),
   **CI drift** (scheduled or hidden `apply` diverging from what's reviewed), or **state
   corruption risk** (manual state edits, missing lifecycle protections on stateful resources).
2. Check **module structure**: single-responsibility modules, sane variable/output contracts, no
   hidden coupling via `terraform_remote_state` where a proper interface would do.
3. Check **variables**: typed, validated (`validation` blocks) where input mistakes would be
   costly, sensible defaults or explicitly required with no default.
4. Check **outputs**: only what downstream consumers need; `sensitive = true` on anything secret.
5. Check **provider constraints**: version pinned with a sensible range, not fully open or fully
   pinned to an exact patch that blocks security updates.
6. Check for **hardcoded values/secrets**: credentials, account IDs, or environment-specific
   values that should be variables or come from a secrets manager.
7. Check **lifecycle rules**: `prevent_destroy` on genuinely stateful/critical resources,
   `create_before_destroy` where replacement would otherwise cause an outage, `ignore_changes`
   used narrowly (not masking real drift).
8. Check `plan` output (if provided) for unexpected **replacements** (`-/+`) versus in-place
   updates — a replacement on a stateful resource (database, volume) is a red flag requiring
   explicit confirmation.
9. Check for **destroy risk**: never treat `terraform destroy`/`-auto-approve` as safe to
   run automatically; if a destroy is genuinely needed, require a `plan -destroy` review first.

## References

- `references/version-guards.md` — version floors for features you might recommend (native
  tests 1.6+, mocks 1.7+, moved/import blocks), Terraform vs OpenTofu divergence, the stuck
  state-lock protocol, and CI-vs-local version-skew diagnosis.
- `references/examples.md` — bad → good Terraform pairs mapped to the risk categories above.
- `references/tools.md` — optional local tools (`tflint`, `checkov`, `trivy config`,
  `gitleaks`) that can supplement this manual review. Never install these automatically.
- `references/state-operations.md` — state surgery decision table (moved/import/removed blocks
  vs CLI), backend migration, state splitting, drift reconciliation, disaster recovery,
  workspaces-vs-directories, and the safe-destroy protocol.
- `references/module-design.md` — module sizing/composition, variable and output contracts,
  version pinning, for_each/count keying rules, dynamic blocks, and named anti-patterns
  (god modules, thin wrappers, hardcoded env assumptions).
- `references/security-compliance.md` — what actually lands in state (write-only/ephemeral
  floors), OIDC vs static credentials, state backend encryption/hardening, policy-as-code
  pipeline placement, and compliance checklist items.

## Safety rules

- Do not run `terraform apply`, `destroy`, or `import` — plan/read-only commands
  (`terraform plan`, `validate`, `fmt -check`, `show`) only, and only when explicitly requested.
- Never run `terraform apply -auto-approve` or `destroy -auto-approve` under any circumstance.
- Never echo secret values found in `.tfvars`, state, or plan output — describe the exposure
  without quoting the value.
- State every finding's risk category explicitly (identity churn / secrets / blast radius / CI
  drift / state corruption) so the user can prioritize.

## Output format

```
## Assumptions
<Terraform/OpenTofu version, provider versions, what wasn't verifiable>

## Risk category: <identity churn | secrets | blast radius | CI drift | state corruption | none>

## Findings
- <finding> — risk: <what breaks / blast radius> — fix: <concrete change>

## Remediation tradeoffs
<if the fix has a cost — e.g. forces a resource replacement — state it explicitly>

## Validation plan
<terraform plan / validate steps to confirm the fix before apply>

## Rollback notes
<how to revert if the change causes a problem after apply>
```

## Quality checklist

- [ ] Risk category is stated explicitly, not left implicit
- [ ] Any finding involving a stateful resource replacement is flagged as high risk
- [ ] No `apply`/`destroy` was run, and none was suggested with `-auto-approve`
- [ ] Secret values were never echoed, only described
- [ ] A validation plan and rollback notes are included, not just the fix
