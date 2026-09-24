---
name: gitops-review
description: Use when reviewing GitOps repository structure, environment separation, ownership, or promotion flow, independent of the controller. Mention "gitops review", "deploy repo structure", or "gitops repo layout"; use argocd for controller operation, argocd-debug for one stuck Application, and argocd-applicationset for generator design.
license: MIT
compatibility: Works with Claude Code, Codex-style agents, CodeWhale, OpenCode, and other agents that support Agent Skills-style instructions.
---

# GitOps Review

## TL;DR checklist

- [ ] Trace where an Application is defined and how a commit reaches each environment.
- [ ] Check secrets, image updates, drift handling, rollback, and bootstrap.
- [ ] Report structural gaps with the affected path and promotion risk.

## Key read-only checks

- Read the repository tree, environment boundaries, application definitions, and pre-merge checks.

## Common pitfalls

- Do not mistake a single manifest finding for a repository-flow finding.

## Agent procedure

Follow the [Workflow](#workflow), [Safety rules](#safety-rules), and [Quality checklist](#quality-checklist) below.

## Quick references

- [references/validation-policy.md](references/validation-policy.md)
- [references/tools.md](references/tools.md)

Last verified: unverified

## When to use

Use when reviewing how a GitOps repository is organized and how changes flow from commit to
running cluster — layout, environment separation, secrets, and promotion process.
Use `argocd` instead for controller operation, `argocd-debug` for one stuck Application, and
`argocd-applicationset` for generator/template design.

## Goal

Find structural gaps in the GitOps repo/process that cause drift, unsafe promotions, or unclear
ownership — not a review of any single manifest.

## Workflow

1. **Layout** — is the repo organized as app-of-apps, ApplicationSet-driven, or a flat list of
   Applications? Is the pattern consistent, or mixed in ways that make it unclear where a given
   app is defined?
2. **Environment separation** — are dev/staging/prod separated by directory, branch, or repo? Is
   it possible to accidentally promote a change to prod by editing the wrong directory, or does
   the structure prevent that class of mistake?
3. **Secrets strategy** — how are secrets handled (Sealed Secrets, External Secrets Operator,
   SOPS, Vault)? Confirm no plaintext secrets are committed, and that the secrets tool used is
   consistent across the repo.
4. **Image update flow** — how do new image versions get into the repo (manual PR, image
   updater/automation)? Is there a review gate before deploy, or does an image push go straight
   to prod?
5. **Drift handling** — is there automated drift detection/self-heal, and is it safe for every
   environment (e.g. `selfHeal: true` might be desired in dev but risky if it silently reverts an
   intentional emergency `kubectl` change in prod without alerting anyone)?
6. **Rollback strategy** — is rollback "revert the Git commit" (fast, auditable) or does it
   require manual cluster intervention? Is the revert path documented?
7. **Policy checks** — are manifests validated pre-merge (schema validation, OPA/Kyverno policy
   checks, `conftest`) or only after they've already synced to a cluster?
8. **Promotion process** — how does a change move dev → staging → prod? Is it a PR/merge between
   directories or branches, and is there a required approval step before prod?
9. **Bootstrap safety** — how is a new cluster/environment bootstrapped into this GitOps setup?
   Is the bootstrap process itself version-controlled and repeatable, or manual/tribal knowledge?

## References

- `references/validation-policy.md` — validation pipeline order (schema → lint → policy →
  cluster dry-run), Kyverno/Gatekeeper policy examples, and Kustomize overlay structure
  patterns and pitfalls (patch ordering, base drift, components).
- `references/tools.md` — optional local tools (`kustomize build`, `conftest`, `yamllint`,
  `gitleaks`) that can supplement this manual review. Never install these automatically.

## Safety rules

- This is a review skill: do not modify the GitOps repo's deployed state or trigger a sync.
- Treat "no automated policy checks pre-merge" as a real risk to call out, not just a
  nice-to-have.
- Flag any path where a single commit can silently reach production with no review gate.

## Output format

```
## Repo layout assessment
## Environment separation issues
## Secrets strategy issues
## Image update / promotion flow issues
## Drift handling / rollback issues
## Recommended changes, prioritized
```

## Quality checklist

- [ ] Environment-separation mistakes (accidental prod promotion) were explicitly considered
- [ ] Secrets strategy was verified, not assumed safe
- [ ] Rollback path (git revert vs. manual) was checked
- [ ] Pre-merge policy validation presence was checked
- [ ] No sync or deploy action was taken
