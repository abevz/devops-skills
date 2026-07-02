# External Secrets Operator deployment

Self-authored reference (no third-party source). ESO architecture, install, the provider-auth
decision that makes or breaks the security model, resource design, rotation, and GitOps
integration. Deployment-planning: propose manifests/values, never apply to a live cluster and
never read secret values.

## Architecture and CRDs

ESO is a controller that reconciles external-manager secrets into Kubernetes `Secret` objects.
CRDs:

| CRD | Scope | Purpose |
|---|---|---|
| `SecretStore` | Namespaced | Connection + auth to one provider, usable by ExternalSecrets in that namespace |
| `ClusterSecretStore` | Cluster | One store shared across namespaces (central team pattern) |
| `ExternalSecret` | Namespaced | "Pull these keys from that store into this Secret" |
| `ClusterExternalSecret` | Cluster | Fan one ExternalSecret across many namespaces by selector |
| `PushSecret` | Namespaced | Reverse: push a k8s Secret *into* the provider |

Install via the `external-secrets/external-secrets` Helm chart. `installCRDs: true` on first
install; on upgrades treat CRDs as a deliberate step (chart-managed CRDs don't always upgrade
in place). Components: the controller, a webhook (validates CRs), and a cert-controller — all
Deployments, HA-capable with `replicaCount`.

## Provider auth — the decision that matters most

The store needs credentials to the external manager. **Keyless workload identity beats a static
credential every time** — a static cloud key stored as a Kubernetes Secret to bootstrap the
thing that manages Kubernetes Secrets is the chicken-and-egg that guts the model.

```yaml
# AWS via IRSA (no static keys) — ServiceAccount annotated with the role, referenced by the store
apiVersion: external-secrets.io/v1
kind: ClusterSecretStore
metadata: {name: aws-sm}
spec:
  provider:
    aws:
      service: SecretsManager
      region: eu-central-1
      auth:
        jwt:
          serviceAccountRef: {name: eso-sa, namespace: external-secrets}   # SA has IRSA role
```

| Cloud | Keyless method | Bind |
|---|---|---|
| AWS | IRSA (OIDC → IAM role) | SA annotation `eks.amazonaws.com/role-arn`; role policy scoped to secret ARNs |
| GCP | Workload Identity | SA annotation `iam.gke.io/gcp-service-account`; IAM `secretmanager.secretAccessor` on specific secrets |
| Azure | Workload Identity | SA annotation `azure.workload.identity/client-id`; access policy on the Key Vault |
| Vault | Kubernetes auth | Vault role bound to the SA; policy scoped to the KV path |

- ❌ `secretRef` to a static access-key Secret when the platform supports workload identity.
- ✅ Scope the role/policy to the exact secret paths ESO serves — not `secretsmanager:*`. A
  compromised ESO SA can read everything the store's identity can.
- `ClusterSecretStore` centralizes but widens blast radius (one identity reads many namespaces'
  secrets); namespaced `SecretStore` per team is tighter but more objects. Choose deliberately.

## ExternalSecret design

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata: {name: db-creds, namespace: payments}
spec:
  refreshInterval: 1h                       # provider poll cadence — match rotation frequency
  secretStoreRef: {name: aws-sm, kind: ClusterSecretStore}
  target:
    name: db-creds                          # the k8s Secret ESO creates
    creationPolicy: Owner                   # ESO owns/GC's it; Merge to co-exist with others
    template:                               # shape the output (combine keys, render config files)
      data:
        DATABASE_URL: "postgres://{{ .user }}:{{ .pass }}@db:5432/app"
  data:
    - {secretKey: user, remoteRef: {key: prod/db, property: username}}
    - {secretKey: pass, remoteRef: {key: prod/db, property: password}}
  # dataFrom: [{extract: {key: prod/db}}]   # pull ALL keys from one remote secret
```

- `refreshInterval`: too low hammers the provider (cost + rate limits); too high misses
  rotations. Match the actual rotation cadence; `0` disables refresh (pull once).
- `creationPolicy`: `Owner` (default, ESO manages lifecycle) vs `Merge` (add keys to an existing
  Secret) vs `Orphan` (don't GC on delete).
- `template` is where you avoid app changes — render a connection string or a whole config file
  from separate provider keys, so the app reads one Secret it already understands.

## Rotation — trace it end to end

Provider rotates → ESO re-syncs on `refreshInterval` → Secret object updates → **the pod must
consume the new value**. That last hop is where rotation silently dies:

| Consumption | Picks up rotation? |
|---|---|
| `env` from secretKeyRef | ❌ read once at container start — needs a pod restart |
| Mounted volume (`secret`) | ✅ kubelet updates the file (~1 min), but the app must re-read it |
| Mounted + app watches file | ✅ true hot rotation |
| Mounted + no re-read | ❌ file changes, app holds the old value in memory |

Wire a reloader (e.g. stakater/Reloader annotation) to roll the Deployment when the Secret
changes, or design the app to re-read. State the chosen mechanism — "ESO syncs it" is only half
the path.

## PushSecret (reverse flow)

`PushSecret` writes a k8s Secret back into the provider — for secrets *generated in-cluster*
(an operator-created password) that should live in the manager as source of truth. Rare; flag
if used to paper over "we created a secret by hand in the cluster."

## GitOps integration

- `ClusterSecretStore`/`ExternalSecret` contain **references, not values** → safe in git.
- Ordering: the store (and its SA/identity) must exist before ExternalSecrets that reference it —
  ArgoCD sync-waves (store early, ExternalSecrets later, apps last) or Flux dependsOn.
- ArgoCD OutOfSync: the **generated** `Secret` isn't in git, so ArgoCD may flag it. Exclude
  managed Secrets from the Application, or mark the ExternalSecret's owned Secret as not-pruned.
  (Cross-ref `argocd-debug` OutOfSync playbook.)
- The ESO install itself lives in the GitOps repo like any platform component.

## Comparison quick-take

| | ESO | Sealed Secrets | SOPS |
|---|---|---|---|
| Source of truth | External manager | Git (encrypted) | Git (encrypted) |
| Rotation | Live sync from provider | Re-seal + commit | Re-encrypt + commit |
| External dependency | Yes (the manager) | No | No (KMS/age key) |
| Secret in git | Reference only | Encrypted blob | Encrypted blob |
| Best fit | Already have Vault/cloud SM | Fully git-native, no manager | GitOps-pure, per-file |

## Troubleshooting (read-only)

```bash
kubectl get externalsecret -A            # STATUS: SecretSynced = ok
kubectl describe externalsecret <name> -n <ns>   # events name the exact failure
kubectl get clustersecretstore <name> -o jsonpath='{.status.conditions}'   # store reachable/authed?
```

| Symptom | Cause |
|---|---|
| `SecretSyncError` / store `not ready` | Provider auth failing — IRSA/WI role missing or unscoped; Vault role/policy wrong |
| Secret created but empty/partial | `remoteRef.key`/`property` typo, or the store identity lacks access to that path |
| Never refreshes on rotation | `refreshInterval: 0`, or the pod consumes via env (needs reloader/rollout) |
| Works in one namespace, not another | Namespaced `SecretStore` missing there; use `ClusterSecretStore` or replicate |
| ArgoCD perpetual OutOfSync on the Secret | Generated Secret not excluded from the Application |
