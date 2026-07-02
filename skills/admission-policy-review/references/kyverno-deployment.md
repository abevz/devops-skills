# Kyverno deployment and operations

Self-authored reference (no third-party source). How Kyverno runs in a cluster, the install
decisions that determine whether it's secure-but-fragile or available-but-bypassable, and how it
integrates with GitOps. Deployment-planning content: propose values/manifests, never run
`helm install` against a live cluster.

## How Kyverno runs

Kyverno is a set of **admission webhook controllers** plus CRDs — not an agent on nodes (unlike
Falco). Components (each its own Deployment, HA-capable):

| Controller | Job | Failure impact |
|---|---|---|
| admission-controller | Serves validating/mutating webhooks | Down + `Fail` policy = nothing admits |
| background-controller | Applies `generate`/`mutateExisting` to existing resources | Generated resources drift |
| reports-controller | Produces PolicyReports (audit results) | No audit visibility |
| cleanup-controller | `CleanupPolicy` TTL deletion | Cleanup stops |

Install via the `kyverno/kyverno` Helm chart. The single most consequential decision is
**webhook failure policy** and it flows from HA:

```yaml
# values.yaml (sketch)
admissionController:
  replicas: 3                    # HA — required if you want failurePolicy: Fail safely
  container:
    resources:
      requests: {cpu: 100m, memory: 256Mi}
config:
  webhooks:
    - failurePolicy: Fail        # Fail = secure (deny on webhook down); Ignore = available (bypass on down)
  # exclude the namespaces that must never be gated or you brick the cluster:
  resourceFiltersExcludeNamespaces: [kube-system, kyverno]
```

- `failurePolicy: Fail` without HA (single replica) = one Kyverno pod restart freezes **all**
  admission (no pods schedule anywhere the webhook matches). HA (3 replicas + PDB +
  anti-affinity) is the precondition for `Fail`. This is the exact tradeoff `admission-policy-
  review` step 8 flags — it's decided here at install time.
- **Never let Kyverno gate its own namespace or kube-system** — a policy that blocks Kyverno's
  own pods (or CNI/CSI in kube-system) is a cluster-wide deadlock. The exclude list is not
  optional.
- `resourceFilters` also control webhook load: matching every resource kind is a scale problem;
  scope to what policies actually target.

## Webhook scoping and performance

Kyverno auto-configures its webhooks from the policies you install (it only intercepts kinds a
policy references) — so a broad `match` on `*` in one policy widens the webhook for the whole
cluster. Review: does any policy match more kinds than it needs? `mutateExisting`/`generate`
with wide matches drive background-controller load.

Timeouts: webhook `timeoutSeconds` (default 10) — a slow policy (external calls, image
verification fetching signatures) that exceeds it triggers `failurePolicy`. Image-verification
policies especially need registry reachability + credentials or they time out → with `Fail`,
that's an outage. Plan registry access for the Kyverno SA before enabling `verifyImages`.

## CRDs and upgrade discipline

- Kyverno ships CRDs (`ClusterPolicy`, `Policy`, `PolicyException`, `PolicyReport`,
  `CleanupPolicy`, etc.). Helm's CRD handling: chart puts them in a place that does **not**
  auto-upgrade on `helm upgrade` — CRD upgrades are a deliberate step (apply the new CRDs from
  the release, then upgrade the chart). Skipping this is the classic "new policy field ignored
  after upgrade."
- Kyverno minor versions track Kubernetes minors and occasionally change policy API (`v1` →
  `v2beta1` fields). Pin the chart version, read the migration notes on upgrade, and test
  policies against the target version (`kyverno test` / `chainsaw`) before rolling.

## GitOps integration (ArgoCD/Flux)

Policies are just manifests — they belong in the GitOps repo. Two ArgoCD-specific gotchas:

- **PolicyReports are noisy in the diff**: the reports-controller writes report objects
  constantly; if ArgoCD manages the namespace it shows perpetual OutOfSync. Add
  `PolicyReport`/`ClusterPolicyReport` (and Kyverno's `AdmissionReport`/`BackgroundScanReport`
  intermediate resources) to `resource.exclusions` in the argocd-cm, or scope the Application to
  exclude them. (Cross-ref `argocd-debug` OutOfSync playbook.)
- **Mutation vs GitOps drift**: a Kyverno `mutate` policy that changes admitted resources makes
  the live object differ from git → ArgoCD OutOfSync forever. Either use `mutate` only on things
  git doesn't manage, or set ArgoCD `ignoreDifferences` for the mutated paths, or move the
  mutation into git instead. Decide deliberately — `verifyImages: mutateDigest: true` has this
  exact effect (rewrites tag→digest), so pair it with digests-in-git (see
  `supply-chain-security`).

Bootstrap ordering: Kyverno (and its CRDs) must sync **before** any policy or any workload the
policy gates — use ArgoCD sync-waves (Kyverno install in an early wave, policies later, apps
last) or the app-of-apps ordering. A policy syncing before its CRD exists = sync error.

## Rollout and verification (read-only)

```bash
# Is admission actually healthy? (webhook down + Fail = cluster-wide admission outage)
kubectl get deploy -n kyverno
kubectl get validatingwebhookconfigurations,mutatingwebhookconfigurations | grep kyverno

# Effective failurePolicy per webhook (the availability/security tradeoff, live)
kubectl get validatingwebhookconfigurations -o json | jq -r '.items[] | select(.metadata.name|test("kyverno")) | .webhooks[] | .name + " failurePolicy=" + .failurePolicy'

# Audit results before flipping anything to Enforce (see kyverno-patterns.md for burn-down)
kubectl get clusterpolicyreport -o json | jq -r '[.items[].results[]?|select(.result=="fail")]|length'
```

Sequencing recap: install HA + exclude system namespaces → policies in `Audit` → burn down
PolicyReport violations → flip per-policy to `Enforce` (prod last) → alert on the Kyverno
Deployment's own availability (a silently-crashed admission-controller with `Ignore` policy =
your policies are off and nothing tells you).

## Deployment troubleshooting

| Symptom | Cause / check |
|---|---|
| Everything stops scheduling after install | `failurePolicy: Fail` + single replica restarted, or a policy gates kube-system/kyverno itself |
| New policy field silently ignored | CRDs not upgraded with the chart |
| Policies don't fire at all | Webhook not registered (controller crashloop), or resource excluded by `resourceFilters` |
| ArgoCD perpetual OutOfSync | PolicyReport objects or mutate-policy drift not excluded |
| Image-verify policy causes intermittent deny | Webhook timeout: registry unreachable / missing creds on Kyverno SA |
| High API latency cluster-wide | Webhook matching too many kinds; scope policy `match` blocks |
