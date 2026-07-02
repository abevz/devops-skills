---
name: helm-review
description: Use when reviewing a Helm chart's templates, values, or upgrade safety. Mention "review this helm chart", "helm chart review", "chart upgrade safety" as triggers.
license: MIT
compatibility: Works with Claude Code, Codex-style agents, CodeWhale, OpenCode, and other agents that support Agent Skills-style instructions.
---

# Helm Review

## When to use

Use when reviewing a Helm chart (own or third-party) before adopting it, upgrading it, or
changing its templates/values.

## Goal

Catch chart-authoring mistakes that cause failed installs, unsafe upgrades, or unpredictable
rendered output across environments.

## Workflow

1. **values schema** — is there a `values.schema.json` or documented contract? Are required
   values validated, or does a missing value silently render broken YAML?
2. **templates** — check for hardcoded values that should be templated, and templated values that
   should be constants; check `{{- if }}`/`{{- with }}` guard correctness (nil pointer panics on
   missing nested values).
3. **helpers (`_helpers.tpl`)** — naming helpers used consistently (no manual string
   concatenation duplicating `fullname`/`name` logic elsewhere).
4. **naming/labels** — standard Helm labels (`app.kubernetes.io/*`) present and consistent across
   all rendered resources, so `helm uninstall` and selectors work correctly.
5. **upgrade safety** — anything that forces resource recreation on every upgrade (e.g.
   immutable field changes, name templated from a value likely to change)? Are Deployment
   rollout settings preserved across upgrades?
6. **hooks** — pre/post-install/upgrade hooks reviewed for idempotency (safe to run on every
   upgrade) and correct `hook-weight`/`hook-delete-policy`.
7. **CRDs** — chart's CRD strategy is understood (Helm does not upgrade or delete CRDs in
   `crds/` automatically) — is that documented/handled?
8. **default values** — defaults are safe for a fresh install (no accidental `replicaCount: 0`,
   no privileged securityContext by default, resources set even if minimal).
9. **secrets** — chart does not template plaintext secrets into values files that get committed;
   integrates with the project's actual secrets strategy (external-secrets, SOPS, sealed-secrets).
10. **tests** — `helm test` hooks or template unit tests (e.g. `helm-unittest`) present for
    critical logic (conditionals, loops).
11. **GitOps compatibility** — chart renders deterministically (no reliance on `Values.random`,
    timestamps, or `lookup` in ways that break `argocd diff`/drift detection).

## References

- `references/patterns.md` — Chart.yaml/values.yaml contracts, template safety rules
  (whitespace/nindent/quote, required helpers, selector immutability), Capabilities-based
  apiVersion branching, and upgrade-behavior pitfalls.
- `references/tools.md` — optional local tools (`helm lint`, `helm template` piped into
  kubeconform/kube-score/polaris/pluto) that can supplement this manual review, all cluster-safe.

## Safety rules

- This is a review skill: do not run `helm install`/`upgrade`/`uninstall` against a live cluster.
- `helm template`/`helm lint` (local, no cluster mutation) are fine to run if available; anything
  touching a live release requires explicit user confirmation.
- Flag non-deterministic rendering as a GitOps-blocking issue, not just a style note.

## Output format

```
## Blocking (breaks install/upgrade or is unsafe)
## Should fix (works but fragile)
## Style / minor
## GitOps compatibility notes
```

## Quality checklist

- [ ] Upgrade safety was checked, not just fresh-install behavior
- [ ] Hooks were checked for idempotency
- [ ] Default values were checked for safe-by-default behavior
- [ ] Non-deterministic rendering was explicitly checked
- [ ] No `helm install`/`upgrade`/`uninstall` was run against a live cluster
