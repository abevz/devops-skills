# Argo CD SSO, RBAC, and AppProject

Self-authored reference (no third-party source). Identity and the tenancy boundary. Propose
config; RBAC/project mistakes lock users out or widen access — frame as reviewed changes.

## SSO — Dex vs direct OIDC

| Approach | Use when |
|---|---|
| Bundled Dex | Your IdP is SAML/LDAP/GitHub/GitLab or needs connector translation to OIDC |
| Direct OIDC (`oidc.config`) | Your IdP speaks OIDC directly (Okta, Keycloak, Google, Entra) — one less component |

```yaml
# argocd-cm (direct OIDC)
oidc.config: |
  name: SSO
  issuer: https://idp.example.com
  clientID: argocd
  clientSecret: $oidc.clientSecret        # $ = pulled from argocd-secret, NOT inline
  requestedScopes: [openid, profile, email, groups]
  requestedIDTokenClaims: {groups: {essential: true}}
```

- The **`groups` claim is what RBAC binds to** — if the IdP doesn't emit groups, RBAC by group
  can't work; verify the claim is present (decode a test token).
- Client secret comes from `argocd-secret` via `$name` reference — never inline in the ConfigMap
  (which is often in git). See `secrets-management`.
- Disable/limit local `admin` once SSO works (`admin.enabled: false` in argocd-cm) — a shared
  admin password is the usual audit finding.

## RBAC model

`argocd-rbac-cm` holds `policy.csv` (Casbin). Two statement types:

```csv
# p = permission: role, resource, action, object, effect
p, role:dev, applications, get, team-a/*, allow
p, role:dev, applications, sync, team-a/*, allow
p, role:dev, applications, delete, team-a/*, deny
# g = group binding: subject → role (or role → role)
g, team-a-oidc-group, role:dev
policy.default: role:readonly           # what unmatched users get — NOT role:admin
```

- Objects are `project/application` (or `*/*`); scope roles to a project, not `*/*`.
- Built-in `role:admin` (everything) and `role:readonly`. `policy.default` should be `readonly`
  or empty — defaulting to admin is a critical misconfig.
- Resources: `applications`, `applicationsets`, `repositories`, `clusters`, `projects`,
  `accounts`, `certificates`, `logs`, `exec` (!). `exec` grants pod shells via the UI — treat as
  privileged, grant narrowly.
- Actions include `sync`, `override`, `action/*` — a user with `sync` but not `override` can
  sync but not force; design the split deliberately.

## AppProject — the real tenancy boundary

RBAC says who can act; the AppProject says what an Application may *do*, regardless of who owns
it. This is the containment when something goes wrong:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata: {name: team-a, namespace: argocd}
spec:
  sourceRepos: ["https://github.com/org/team-a-*"]     # only these repos
  destinations:
    - {server: https://kubernetes.default.svc, namespace: "team-a-*"}   # only these ns
  clusterResourceWhitelist: []                          # no cluster-scoped resources by default
  namespaceResourceBlacklist:
    - {group: "", kind: ResourceQuota}                  # tenants can't edit their own quota
  roles:                                                # project-scoped tokens
    - {name: ci, policies: ["p, proj:team-a:ci, applications, sync, team-a/*, allow"]}
  syncWindows:
    - {kind: deny, schedule: "0 22 * * *", duration: 8h, applications: ["*"]}  # no syncs overnight
```

- `default` project allows everything (`*` repos/destinations/cluster resources) — **never leave
  tenant apps in `default`**; a compromised or buggy app in `default` can deploy cluster-scoped
  resources anywhere. This is the #1 Argo CD tenancy finding.
- `clusterResourceWhitelist: []` (empty) = tenants can't create ClusterRoles/CRDs/etc. — the safe
  default; widen per explicit need.
- `syncWindows` gate when syncs happen (change-freeze windows, business hours) per project.
- Project `roles` issue project-scoped tokens for CI — scoped to that project's apps only, better
  than a global token.

## Verify (read-only)

```bash
argocd proj get team-a
argocd proj role list team-a
kubectl get configmap argocd-rbac-cm -n argocd -o jsonpath='{.data.policy\.default}'   # must not be role:admin
argocd account can-i sync applications team-a/myapp --auth-token <token>   # test effective RBAC
```
