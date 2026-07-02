---
name: argocd
description: Use when installing, configuring, or operating Argo CD itself — HA deployment, controller sharding, SSO/RBAC, AppProject design, repository/credential setup, sync policy design, config management plugins, disaster recovery, and scaling. Mention "install argocd", "argocd HA", "argocd rbac", "argocd sso", "appproject", "argocd backup", "argocd sharding" as triggers.
license: MIT
compatibility: Works with Claude Code, Codex-style agents, CodeWhale, OpenCode, and other agents that support Agent Skills-style instructions.
---

# Argo CD

## When to use

Use when standing up or operating Argo CD as a platform: HA install, scaling/sharding, SSO and
RBAC, AppProject boundaries, connecting repositories and credentials, designing sync policy,
config-management plugins, backup/DR, and upgrades. For troubleshooting a single out-of-sync
Application use `argocd-debug`; for ApplicationSet generator design use `argocd-applicationset`;
for the GitOps repo layout and promotion flow use `gitops-review`. This skill is the operator/
platform view of Argo CD itself.

## Goal

A correctly-sized, secure, recoverable Argo CD install: HA where it matters, RBAC/projects that
actually contain tenants, repositories connected without plaintext secrets, sync behavior chosen
deliberately per environment, and a tested way to rebuild it.

## Workflow

1. **Scope to an area** and load its reference:

   | Area | Reference |
   |---|---|
   | Install, HA topology, components, controller sharding, repo-server scaling, upgrades | `references/install-ha.md` |
   | SSO (Dex/OIDC), RBAC model, AppProject as security boundary, project-scoped tokens, sync windows | `references/rbac-sso.md` |
   | Repository connections, credential types, private/OCI registries, config-management plugins, secret injection | `references/repositories.md` |
   | Application/sync design: syncPolicy, sync options, waves, hooks, tracking, health, ignoreDifferences | `references/sync-design.md` |
   | Backup/DR, notifications, metrics/monitoring, performance tuning, multi-tenancy | `references/operations.md` |

2. **Confirm the baseline** — Argo CD version, install method (Helm `argo-cd` chart vs raw
   manifests), single vs HA, how many clusters/apps it manages (drives sharding and repo-server
   sizing), and where it authenticates users from.
3. **Right-size, don't over-build** — a single-instance Argo CD is fine for a small setup; HA
   (redis-ha, controller sharding, multi-replica repo-server) is warranted by app/cluster count
   and blast-radius requirements. Recommend HA against a real need, not reflexively.
4. **Security first-class** — SSO + RBAC + AppProject are the tenancy boundary; a shared admin
   Argo CD with no projects is a foot-gun. Treat the `argocd-server` and its RBAC like any other
   privileged control plane.
5. **Make it recoverable** — Argo CD is declarative, but its repo/cluster credentials and RBAC
   live in k8s Secrets/ConfigMaps; DR means both "everything in git" AND those Secrets backed up.
6. **GitOps the platform itself** — Argo CD config (ConfigMaps, projects, repos, RBAC) is
   declarative and belongs in git (often an app-of-apps managing Argo CD) — not clicked in the UI.
7. **Verify read-only** — `argocd admin settings`, `kubectl get` on argocd resources,
   `argocd proj get`, metrics; never change live config as "investigation."

## References

- `references/install-ha.md`, `references/rbac-sso.md`, `references/repositories.md`,
  `references/sync-design.md`, `references/operations.md` — one per area, each with decision
  tables, Helm/CRD/config examples, prerequisites, and gotchas.

## Safety rules

- Design/operate-planning skill: do not `argocd app sync/delete`, apply projects/RBAC, rotate
  credentials, or run `argocd admin import` against a live install — propose manifests/values.
- RBAC and AppProject changes can lock users out or widen access cluster-wide — frame them as
  reviewed, reversible changes; never loosen a project to `*` reflexively.
- Never put repository/cluster credentials or SSO client secrets in plaintext in git — reference
  a secrets mechanism (see `secrets-management`); never echo a token/secret value.
- DR/import operations overwrite state — treat `argocd admin import`, credential rotation, and
  redis flushes as destructive, requiring explicit confirmation.

## Output format

```
## Area & baseline (version, install method, HA?, #clusters/#apps, auth source)
## Recommendation (what to configure/change, and explicitly what NOT to over-build)
## Prerequisites & blast radius
## Proposed config (Helm values / projects / RBAC / repo secrets — for review)
## Rollout & rollback
## DR/verification (read-only checks, backup coverage)
```

## Quality checklist

- [ ] The relevant area reference was consulted, not reasoned from memory
- [ ] HA/sharding recommended against actual app/cluster scale, not reflexively
- [ ] AppProject + RBAC form a real tenancy boundary, not a shared-admin free-for-all
- [ ] Credentials/secrets are referenced via a secrets mechanism, never plaintext in git
- [ ] DR covers both git state AND the Secrets/ConfigMaps Argo CD needs to rebuild
- [ ] Nothing was synced/applied/imported on a live install
