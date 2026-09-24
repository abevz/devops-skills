---
name: secrets-management
description: Use when designing, deploying, or reviewing Kubernetes secrets management — External Secrets Operator, Sealed Secrets, SOPS, or Vault. Mention "external secrets", "ESO", "sealed secrets", "SOPS", "how do we manage secrets", "secret rotation" as triggers.
license: MIT
compatibility: Works with Claude Code, Codex-style agents, CodeWhale, OpenCode, and other agents that support Agent Skills-style instructions.
---

# Secrets Management

## TL;DR checklist

- [ ] Choose the source of truth: external manager, encrypted Git, or runtime mount.
- [ ] Check provider authentication, secret scope, rotation, and consumer behavior.
- [ ] Keep plaintext secret material out of Git and review the delivery path.

## Key read-only checks

- Inspect SecretStore/ExternalSecret or encrypted-file configuration and the workload reference; do not print secret values.

## Common pitfalls

- Do not hide a static credential inside Kubernetes merely to bootstrap a secret operator.

## Agent procedure

Follow the [Workflow](#workflow), [Safety rules](#safety-rules), and [Quality checklist](#quality-checklist) below.

## Quick references

- [references/eso-deployment.md](references/eso-deployment.md)

Last verified: unverified

## When to use

Use when choosing a Kubernetes secrets approach, **deploying** External Secrets Operator (or
Sealed Secrets / SOPS / Vault), designing `ExternalSecret`/`ClusterSecretStore` resources,
planning rotation, or reviewing how secrets flow from a source of truth to running pods.
Complements `gitops-review` (where secrets fit in the repo) and `kubernetes-security` (secret
exposure at the pod).

## Goal

Secrets never live in git as plaintext, come from a real source of truth, are rotatable, and are
readable only by the workloads that need them — with the operator itself deployed so it can't
become the weak link.

## Workflow

0. **Choose the approach** — the first decision, driven by where the source of truth is:

   | Approach | Source of truth | Best when |
   |---|---|---|
   | External Secrets Operator (ESO) | External manager (AWS/GCP/Azure SM, Vault, etc.) | You already run a cloud/Vault secret store; want live sync + rotation |
   | Sealed Secrets | Git (encrypted with a cluster key) | No external manager; want fully git-native secrets |
   | SOPS (+ age/KMS) | Git (encrypted files) | GitOps-pure, per-file encryption, Flux/ArgoCD plugin decrypts at apply |
   | Vault Agent / Secrets Store CSI | Vault / cloud store, mounted at runtime | Secrets should never become k8s Secrets at all (mounted files/env only) |

   ❌ committing a plaintext `Secret` manifest. ❌ mixing three approaches without a reason.

1. **Deploy the operator securely** (if not running) — for ESO see `references/eso-deployment.md`:
   controller install, the provider-auth decision (workload identity vs static creds — the crux),
   `ClusterSecretStore` vs namespaced `SecretStore`, least-privilege on the store's credentials.
2. **Provider authentication** — the security center of gravity. Prefer keyless workload
   identity (IRSA / GKE Workload Identity / AKS Workload Identity) over a static credential
   stored... as a Kubernetes Secret (the chicken-and-egg that undermines the whole point). Scope
   the store's access to only the secret paths it serves.
3. **ExternalSecret / consumption design** — `refreshInterval` sane (not 1s hammering the
   provider, not 24h for something rotated hourly), `data` vs `dataFrom` (whole-secret pull),
   target `template` to shape the resulting Secret, and how pods consume it (mounted file >
   env var for high-value secrets, per `kubernetes-security`).
4. **Rotation** — is rotation actually end-to-end? Provider rotates → ESO re-syncs on refresh →
   **the pod must pick up the new value** (env vars are read once at start; mounted secrets
   update but the app must re-read, or a reloader/rollout is needed). A rotation that stops at
   the Secret object and never reaches the running process is not rotation.
5. **Least access & audit** — RBAC on who can read the synced Secrets; the provider's own audit
   log shows who/what pulled each secret; no wildcard `secrets` `get` (see `kubernetes-security`
   RBAC vectors).
6. **GitOps integration** — `ExternalSecret`/`ClusterSecretStore` are plain manifests in git
   (they contain references, not values — safe to commit); ordering so the store exists before
   ExternalSecrets that use it; ArgoCD shows the generated Secret as OutOfSync unless excluded.

## References

- `references/eso-deployment.md` — ESO architecture and CRDs, Helm install, provider-auth
  patterns per cloud (IRSA/Workload Identity/Vault), SecretStore vs ClusterSecretStore,
  ExternalSecret/PushSecret spec, rotation and reloader wiring, GitOps ordering and ArgoCD
  exclusions, comparison with Sealed Secrets/SOPS, and troubleshooting.

For operating Vault itself (HA, seal/auto-unseal, auth methods, policies, secret engines, DR)
rather than consuming its secrets into Kubernetes, use the `vault` skill.

## Safety rules

- Never read, print, or exfiltrate actual secret values — work with references, keys, and
  metadata only. If a task would require dumping a secret's value, stop and explain why not.
- This skill designs/reviews and plans deployment: do not apply SecretStores/ExternalSecrets or
  create provider credentials against a live cluster/account — propose manifests.
- Never propose storing a long-lived provider credential as a Kubernetes Secret when workload
  identity is available — call that out as the anti-pattern it is.
- Provider auth config (roles, IRSA ARNs, Vault roles) is sensitive — reference, don't invent
  account IDs or paste real ARNs.

## Output format

```
## Approach (chosen tool + why, vs alternatives)
## Provider auth (workload identity vs static — the decision)
## Store & ExternalSecret design
## Rotation path (source → Secret → running pod — end to end?)
## Least-access & audit
## GitOps wiring (ordering, ArgoCD exclusions)
## Gaps / risks
```

## Quality checklist

- [ ] Provider auth uses workload identity where available, not a static credential-in-Secret
- [ ] Rotation was traced all the way to the running pod, not just the Secret object
- [ ] The store's access is scoped to the secret paths it serves, no wildcards
- [ ] ExternalSecrets/SecretStores in git contain references only, never values
- [ ] No secret value was read or printed; nothing was applied to a live cluster
