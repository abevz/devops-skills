# Vault auth methods and policies

Self-authored reference (no third-party source). How workloads/humans authenticate and what
they're allowed. Propose config; never issue tokens or read secret values.

## Auth methods — by identity, not shared secrets

| Method | Identity | Use |
|---|---|---|
| Kubernetes | Pod ServiceAccount JWT → Vault role | In-cluster workloads (the default for k8s) |
| JWT/OIDC | CI OIDC token, IdP | CI/CD (GitHub Actions OIDC), human SSO |
| AppRole | role-id + secret-id | Machines outside k8s; secret-id delivery is the weak point |
| Cloud IAM (AWS/GCP/Azure) | Instance/workload identity | Cloud VMs/functions |
| Token | raw token | Bootstrap/break-glass only |
| Userpass | password | ❌ avoid except tiny/dev |

- ✅ Kubernetes auth for pods: Vault verifies the SA JWT against the cluster and maps
  `(SA, namespace)` → a Vault role → policies. No secret to distribute.
- ❌ AppRole with a long-lived `secret-id` stored in a k8s Secret — that's the shared-secret
  anti-pattern Vault exists to remove; use k8s auth instead. If AppRole is unavoidable, use
  response-wrapping / short secret-id TTL / push delivery, never a static committed secret-id.
- Every auth role has bounded token TTL/max-TTL and the minimum policies.

```hcl
# Kubernetes auth role (shape): bind namespace+SA → policy, short TTL
# vault write auth/kubernetes/role/payments \
#   bound_service_account_names=api \
#   bound_service_account_namespaces=payments \
#   policies=payments-read token_ttl=15m token_max_ttl=1h
```

## Policy design (HCL)

Policies are path-based allow rules; default is deny.

```hcl
# payments-read: least privilege, KV v2 (note the /data/ segment)
path "secret/data/payments/*" {
  capabilities = ["read"]
}
path "secret/metadata/payments/*" {      # list needs metadata path on KV v2
  capabilities = ["list"]
}
# dynamic DB creds for this app only
path "database/creds/payments-ro" {
  capabilities = ["read"]
}
```

- Capabilities: `create update read delete list sudo deny`. Grant the few needed, per path.
- **KV v2 path gotcha**: data is at `secret/data/<path>`, metadata/list at `secret/metadata/<path>`
  — a policy written as `secret/<path>` (v1 style) silently grants nothing on a v2 mount. Top
  policy bug.
- `deny` wins over any allow — use for carve-outs.
- ❌ `path "*" { capabilities = ["sudo","read",...] }` or broad `secret/*` — that's cluster-admin
  for secrets. Scope to the app's subtree.

## Templated / ACL policies (per-identity paths)

```hcl
# each entity reads only its own path, via identity templating
path "secret/data/{{identity.entity.aliases.<mount>.metadata.namespace}}/*" {
  capabilities = ["read"]
}
```

Templated policies let one policy serve many tenants by injecting identity metadata — cleaner
than one policy per team, and enforced by Vault identity, not by naming discipline.

## Identity: entities and groups

- Multiple auth aliases (k8s SA + OIDC user) can map to one **entity**; **groups** (internal or
  external, mapped from OIDC `groups`) attach policies — the same group-drives-RBAC model as
  Argo CD/Kubernetes. Bind policies to groups, not individual entities, for humans.

## Verify (read-only)

```bash
vault auth list
vault policy list && vault policy read payments-read
vault read auth/kubernetes/role/payments
vault token lookup                        # current token's policies/TTL (no secret values)
```
