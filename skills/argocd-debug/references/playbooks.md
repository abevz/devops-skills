# ArgoCD failure playbooks

Symptom-keyed diagnosis trees. All diagnosis commands are read-only.

## OutOfSync but "nothing changed in git"

The diff (`argocd app diff <app>`) tells you which of these it is:

| Diff shows | Cause | Fix |
|---|---|---|
| `replicas` differs | HPA (or manual scale) owns replicas, manifest also sets it | Remove `replicas` from the manifest, or add `ignoreDifferences` for `/spec/replicas` |
| Injected containers/labels/annotations you didn't write | Mutating webhook (service mesh sidecar, policy agent, cert-manager) rewrites live objects | `ignoreDifferences` with `jqPathExpressions` for the injected paths, or `RespectIgnoreDifferences=true` in syncOptions |
| Fields with default values (`protocol: TCP`, empty maps, ordering) | Server-side defaulting/normalization the diff engine doesn't normalize | Enable `ServerSideDiff=true` (or upgrade ArgoCD); as a targeted fix, `ignoreDifferences` |
| Secret data differs constantly | Operator/controller rotates the secret (cert renewal etc.) | Stop managing the rotating field; `ignoreDifferences` on the data key |
| Whole objects appear only on one side | `targetRevision` is a moving branch that moved, or another Application/controller also manages the namespace | Pin revisions per environment; check for overlapping Application `spec.source.path`s |

Rule of thumb: persistent OutOfSync with automated sync enabled = something else keeps writing
to the live object. Find the writer (`kubectl get <res> -o yaml` → `metadata.managedFields`
shows the field manager) before adding `ignoreDifferences` — ignoring can mask real drift.

## ComparisonError / manifest generation errors

`argocd app get <app>` → conditions block quotes the error:

| Condition/message contains | Cause | Fix |
|---|---|---|
| `authentication required` / `permission denied` | Repo credentials wrong or missing | Check the repo Secret (label `argocd.argoproj.io/secret-type: repository`), token expiry, and URL match (SSH vs HTTPS forms are different repo entries) |
| `app path does not exist` | `spec.source.path` moved/renamed | Fix the path; grep git history for the rename |
| `helm dependency` / `failed to fetch chart` | Chart dependency repos unreachable or OCI login missing | Reproduce locally: `helm dependency build`; add missing repos/creds to ArgoCD |
| `failed exit status 1` from kustomize/helm | Template/overlay renders differently or fails in ArgoCD's env | Reproduce with the **same binary version** ArgoCD uses (`argocd version`, tool versions in argocd-cm) — version skew between laptop and repo-server is the classic cause |
| `rpc error ... deadline exceeded` | Repo-server timeout: huge repo, monorepo without path filtering, slow git host | Shallow/filtered fetch settings, `webhook` instead of polling, split the repo source |

## App Degraded / Progressing forever

Health is aggregated from the resource tree — find the leaf, not the app:

1. `argocd app get <app>` → the resource list marks which object is Degraded/Progressing.
2. Route by that object's kind:
   - **Deployment Progressing=False** (`ProgressDeadlineExceeded`) → its pods are the problem →
     switch to the `kubernetes-debug` skill playbooks (CrashLoop, ImagePull, Pending).
   - **PVC Pending** → StorageClass/provisioner (kubernetes-debug → Pending).
   - **Job failed** → `kubectl logs job/<name>`; if it's a hook Job, see Hooks below.
   - **Custom resource stuck Progressing** → its health check: ArgoCD may lack a health Lua for
     this CRD (shows Progressing forever despite being fine) → custom health check in argocd-cm.
3. App `Healthy` in ArgoCD ≠ app actually works — health checks only see K8s status fields.

## Sync starts but never finishes / hangs

- **PreSync hook failing**: sync waits on hook Jobs. `kubectl get jobs -n <ns>` → failed
  migration/init Job blocks everything. Old hook Jobs with `hook-delete-policy` unset can also
  collide on immutable fields (Job spec) — set `HookSucceeded`/`BeforeHookCreation`.
- **Sync waves stuck**: a resource in wave N unhealthy blocks wave N+1. Order matters: CRDs and
  namespaces in early (negative) waves, CRs later. `argocd.argoproj.io/sync-wave` annotations.
- **Pruning waiting on finalizers**: object stuck `Terminating` (finalizer of a dead controller)
  blocks prune — identify the finalizer via `kubectl get <res> -o yaml`; removing finalizers is
  a mutation requiring explicit user confirmation and understanding of why it's stuck.

## SharedResourceWarning / resource fights

Two Applications (or an ApplicationSet template overlap) claim the same object — conditions show
`SharedResourceWarning`. Diagnose with the `argocd.argoproj.io/tracking-id` annotation on the
live object. Fix ownership (one app per object), don't suppress the warning.

## Sync fails with permission errors

- `... is forbidden: cannot create resource ...` from the **Application controller** → the
  destination cluster's ArgoCD service account / cluster secret lacks RBAC for that
  kind/namespace.
- `application ... is not permitted in project` → `AppProject` restrictions
  (sourceRepos/destinations/clusterResourceWhitelist) — intentional guardrail; extend the
  project deliberately, not to `*` reflexively.

## Dangerous states to flag (never auto-fix)

- Diff shows mass deletions + automated sync with `prune: true` → a path/generator change is
  about to delete real resources. Stop and show the user the prune list.
- `selfHeal: true` reverting emergency manual changes silently during an incident — check
  `argocd app history` timestamps against incident timeline.
