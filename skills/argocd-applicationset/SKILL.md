---
name: argocd-applicationset
description: Use when designing or reviewing an ArgoCD ApplicationSet for multi-cluster or multi-environment deployment. Mention "applicationset", "argocd multi-cluster", "generator design" as triggers.
license: MIT
compatibility: Works with Claude Code, Codex-style agents, CodeWhale, OpenCode, and other agents that support Agent Skills-style instructions.
---

# ArgoCD ApplicationSet

## TL;DR checklist

- [ ] Identify generator inputs and who owns cluster/environment membership.
- [ ] Check template expansion for every generated Application.
- [ ] Review target scope and promotion blast radius.

## Key read-only checks

- Read generator selectors, cluster secrets or Git paths, template values, and generated Application names.

## Common pitfalls

- Do not assume a new cluster or directory cannot expand the generator unexpectedly.

## Agent procedure

Follow the [Workflow](#workflow), [Safety rules](#safety-rules), and [Quality checklist](#quality-checklist) below.

## Quick references

- [references/generators.md](references/generators.md)

Last verified: unverified

## When to use

Use when designing a new ApplicationSet, or reviewing an existing one, for rolling out apps
across multiple clusters or environments.

## Goal

Ensure the ApplicationSet's generator and template produce the right Applications, with a
blast radius and promotion flow that matches how the team actually wants to ship changes.

## Workflow

1. **Generators** — identify which generator(s) are used (`list`, `cluster`, `git`, `matrix`,
   `merge`, `pull request`) and whether the choice matches the actual source of truth for
   cluster/environment membership (hardcoded list vs. cluster secrets vs. repo directory
   structure).
2. **Templates** — check that templated fields (namespace, `targetRevision`, values files,
   destination) resolve correctly for every generated combination, including edge cases (a
   cluster with no matching values file, a new environment added later).
3. **Cluster targeting** — verify the generator can't accidentally target a cluster/environment
   it shouldn't (e.g. a `list` generator with a stale entry, a `cluster` generator with too broad
   a label selector).
4. **Repo structure** — does the Git layout the generator reads from make it obvious which
   environments exist and what's deployed where, without needing to read the ApplicationSet
   itself?
5. **Naming** — generated Application names are unique, stable, and traceable back to
   cluster+app (avoid collisions when adding a new cluster/env).
6. **Sync policy** — is `automated.selfHeal`/`prune` appropriate per environment (e.g. more
   caution in production than in dev)? Is this configurable per-generated-app, not just
   globally?
7. **Blast radius** — if the ApplicationSet controller or generator source is briefly wrong, how
   many clusters/environments get a bad sync simultaneously? Prefer generator designs that allow
   staged rollout (e.g. matrix generator combining an app list with a *staged* environment list)
   over one that syncs every cluster at once.
8. **Promotion flow** — is there a clear path for a change to go dev → staging → prod (e.g. via
   separate branches/directories/`targetRevision` per stage), or does everything track the same
   revision everywhere?
9. **Multi-env safety** — confirm `AppProject` restrictions prevent a generated Application from
   being pointed at a namespace/cluster outside its intended scope.

## References

- `references/generators.md` — generator decision table, staged-rollout revision patterns,
  deletion-protection options (preserveResourcesOnDeletion, create-update policies),
  missingkey=error and other template correctness traps, AppProject fencing, and read-only
  audit one-liners.

For installing/operating Argo CD itself (HA, sharding, SSO/RBAC, repo setup, backup/DR) use the
`argocd` skill; for single-Application troubleshooting use `argocd-debug`.

## Safety rules

- This is a design/review skill: do not create, sync, or delete ApplicationSets or the
  Applications they generate on a live cluster.
- Flag any design where a single bad commit/generator change could sync all environments
  (including production) simultaneously — this is the primary blast-radius risk for
  ApplicationSets.
- Recommend staged promotion over simultaneous multi-env sync unless the user explicitly wants
  simultaneous rollout.

## Output format

```
## Generator(s) used and why
## Template correctness issues
## Blast radius assessment
## Promotion flow assessment
## Recommended changes
```

## Quality checklist

- [ ] Blast radius (how many envs sync at once on a bad change) was explicitly assessed
- [ ] Promotion flow (dev→staging→prod path) was checked, not assumed
- [ ] Naming collisions across generated Applications were checked
- [ ] AppProject scoping was checked against generator targeting
- [ ] No ApplicationSet/Application was created or synced on a live cluster
