---
name: vault
description: Use when deploying, configuring, or operating HashiCorp Vault (or OpenBao) — HA/storage, seal/auto-unseal, auth methods, policies, secret engines (KV/dynamic/PKI/transit), Kubernetes integration, and DR. Mention "vault", "openbao", "auto-unseal", "vault policy", "dynamic secrets", "vault pki", "vault kubernetes auth" as triggers.
license: MIT
compatibility: Works with Claude Code, Codex-style agents, CodeWhale, OpenCode, and other agents that support Agent Skills-style instructions.
---

# Vault

## When to use

Use when standing up or operating HashiCorp Vault / OpenBao: HA topology and storage, seal /
auto-unseal, auth methods, policy design, secret engines (static KV, dynamic DB/cloud creds,
PKI, transit encryption-as-a-service), Kubernetes integration (k8s auth, Agent injector, Secrets
Store CSI, or External Secrets pulling from Vault), and backup/DR. For *consuming* Vault secrets
into Kubernetes specifically, `secrets-management` covers the ESO-vs-alternatives choice; this
skill is Vault itself.

## Goal

A Vault that is highly available, recoverable, and least-privilege by construction — sealed
safely, authenticating workloads by identity (not shared tokens), issuing the narrowest secrets
possible (prefer dynamic/short-lived over static), with a tested unseal and restore path.

## Workflow

1. **Scope to an area** and load its reference:

   | Area | Reference |
   |---|---|
   | Deploy, HA (Raft/Consul), seal & auto-unseal, upgrades, OpenBao | `references/deploy-ha.md` |
   | Auth methods (k8s, JWT/OIDC, AppRole, cloud IAM) + policy (HCL, path grants, templating) | `references/auth-policies.md` |
   | Secret engines: KV v2, dynamic DB/cloud, PKI, transit; leases & rotation | `references/secret-engines.md` |
   | Kubernetes consumption (k8s auth, Agent injector, CSI, ESO) + backup/DR | `references/kubernetes-and-dr.md` |

2. **Confirm the baseline** — Vault vs OpenBao, version, storage backend (integrated Raft vs
   Consul vs cloud), seal type (auto-unseal configured or manual keys?), HA replica count, and
   how workloads authenticate today.
3. **Least privilege by default** — policies grant the narrowest paths; prefer **dynamic,
   short-lived** secrets (DB/cloud creds that expire) over long-lived static KV where the engine
   supports it; every token/role has a bounded TTL.
4. **Auth by workload identity, not shared secrets** — Kubernetes auth (SA token → Vault role),
   JWT/OIDC, or cloud IAM over a shared AppRole secret-id sitting in a k8s Secret.
5. **Seal safety is the availability story** — a manual-unseal Vault is a 3am-page waiting to
   happen; auto-unseal (cloud KMS/HSM) is the operational default. The unseal/recovery keys are
   the crown jewels — their custody is a documented, split (Shamir) process.
6. **Make it recoverable** — Raft snapshots (or Consul backups) on a schedule, stored off-box,
   and a **tested restore**. Losing Vault storage without a snapshot = every dynamic credential
   and the root of trust gone.
7. **GitOps the config, not the secrets** — policies, auth roles, engine mounts are declarative
   (Terraform Vault provider / vault-operator / config files) and belong in version control; the
   secret *values* never do.
8. **Verify read-only** — `vault status`, `vault policy read`, `vault auth list`,
   `vault secrets list`, `vault token lookup`; never read a secret value or mutate config as
   "investigation."

## References

- `references/deploy-ha.md`, `references/auth-policies.md`, `references/secret-engines.md`,
  `references/kubernetes-and-dr.md` — one per area, with decision tables, config/HCL examples,
  prerequisites, and gotchas.

## Safety rules

- Design/operate-planning skill: do not `vault write/kv put`, apply policies, unseal, rotate
  roots, or run restores against a live Vault — propose config for the user.
- Never read, print, or exfiltrate secret values, tokens, unseal/recovery keys, or root tokens —
  work with paths, policies, and metadata only. If a task requires a secret value, stop and say
  why not.
- Root tokens are for break-glass only — recommend revoking after setup (`vault token revoke`);
  never propose leaving a root token in use or in a file.
- Unseal/recovery keys and auto-unseal KMS access are the highest-value targets — reference their
  custody process, never handle the actual key material.
- Treat unseal, root rotation, and snapshot restore as destructive/critical — explicit
  confirmation, never automatic.

## Output format

```
## Area & baseline (Vault/OpenBao, version, storage, seal type, HA?, auth model)
## Recommendation (what to configure, and explicitly what NOT to over-build)
## Prerequisites & blast radius
## Proposed config (HCL policies / auth roles / engine mounts — for review)
## Rollout & rollback
## DR/verification (snapshot coverage, unseal path, read-only checks)
```

## Quality checklist

- [ ] The relevant area reference was consulted, not reasoned from memory
- [ ] Policies grant least-privilege paths; dynamic/short-lived secrets preferred over static
- [ ] Workloads authenticate by identity (k8s/JWT/cloud IAM), not a shared long-lived secret
- [ ] Auto-unseal + documented key custody, not manual unseal by default
- [ ] Snapshot/restore path is defined and tested; root token revoked after setup
- [ ] No secret/key/token value was read; nothing was written/unsealed on a live Vault
