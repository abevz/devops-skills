---
name: argocd-debug
description: Use when diagnosing one Argo CD Application that is OutOfSync, Degraded, Unknown, or failing to deploy. Mention "argocd is stuck", "application out of sync", or "argocd debug"; use argocd for controller installation/operation, argocd-applicationset for generator design, and gitops-review for repository flow.
license: MIT
compatibility: Works with Claude Code, Codex-style agents, CodeWhale, OpenCode, and other agents that support Agent Skills-style instructions.
---

# ArgoCD Debug

## When to use

Use when an ArgoCD `Application` is `OutOfSync`, `Degraded`, `Unknown`, or stuck, and the cause
needs to be found before taking action.
Use `argocd` instead for controller installation or operation, `argocd-applicationset` for
generator design, and `gitops-review` for repository structure or promotion flow.

## Goal

Diagnose the actual cause of the sync/health problem using read-only inspection, and propose a
fix the user (or ArgoCD's own automated sync) can apply.

## Diagnostic routing

Identify the symptom, then jump to its playbook in `references/playbooks.md`:

| Symptom | Playbook section |
|---|---|
| OutOfSync, "nothing changed in git" | Diff-content → cause table (HPA, webhooks, defaulting, rotation, moving revision) |
| ComparisonError / manifest generation failed | Condition text → cause table (repo auth, path, helm deps, tool version skew) |
| Degraded / Progressing forever | Find the leaf resource, route by kind (→ kubernetes-debug for pods) |
| Sync hangs mid-way | Hooks, sync waves, finalizer-blocked prune |
| SharedResourceWarning | Two apps own one object — tracking-id diagnosis |
| Permission errors on sync | Destination RBAC vs AppProject restrictions |

Use the general workflow below when nothing matches or the picture is unclear.

## Workflow

1. Check `Application` status: `sync status`, `health status`, and `conditions` for explicit
   error messages (`argocd app get <app>` or `kubectl get application <app> -o yaml`).
2. Check the diff between desired and live state (`argocd app diff`) to see exactly what's out of
   sync, or whether `ignoreDifferences` is masking a real drift.
3. Check `app conditions` for repo access errors, manifest generation errors (Helm/Kustomize
   rendering failures), or comparison errors.
4. Verify `targetRevision` and `path` actually point at what's expected — a common cause of
   "nothing changed but it's out of sync" is a moved/renamed path or an unpinned branch that
   moved.
5. If Helm/Kustomize-based, check that the chart/overlay renders locally the same way ArgoCD
   renders it (`helm template` / `kustomize build`) to rule out a rendering-environment
   mismatch.
6. Check `hooks` and `syncPolicy.syncOptions` (e.g. `sync-wave` ordering, `Skip`, `Replace`,
   `PruneLast`) if resources are applying in the wrong order or being skipped.
7. Check destination namespace: does it exist, and does `syncPolicy.automated.selfHeal`/
   `syncPolicy.syncOptions: [CreateNamespace=true]` match expectations?
8. Check ArgoCD's own RBAC/project restrictions (`AppProject`) if the app can't sync due to
   permission errors.
9. Correlate findings into a root cause before proposing a fix.

## References

`references/playbooks.md` — symptom→cause→fix diagnosis trees. For installing/operating Argo CD
itself (HA, sharding, SSO/RBAC, AppProject design, repo setup, backup/DR) use the `argocd` skill.

## Safety rules

- Do not run `argocd app sync`, `argocd app delete`, or any command that changes live cluster
  state unless the user explicitly asks for it.
- Prefer `argocd app diff` / `get` / `kubectl get -o yaml` (read-only) throughout the
  investigation.
- If a force-sync or resource deletion looks like the right fix, say so explicitly and explain
  the blast radius — don't run it.

## Output format

```
## Observations
## Evidence (diff / conditions / logs, quoted)
## Root cause
## Fix
<exact command(s), explicitly marked as requiring user confirmation to run>
## Validation
<how to confirm the fix worked>
```

## Quality checklist

- [ ] Root cause is backed by quoted diff/condition output, not guessed
- [ ] No sync/delete/rollback command was executed without explicit confirmation
- [ ] Rendering (Helm/Kustomize) mismatch was ruled in or out explicitly
- [ ] sync-wave/hook ordering was checked if resource ordering looked wrong
- [ ] AppProject/RBAC restrictions were checked if sync failed for permission reasons
