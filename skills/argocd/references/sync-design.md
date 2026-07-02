# Argo CD Application and sync design

Self-authored reference (no third-party source). Designing sync behavior deliberately per
environment. (Troubleshooting a stuck/out-of-sync app → `argocd-debug`; generator design →
`argocd-applicationset`.) Propose config; don't sync/apply.

## Application source types

| source | Rendered by | Note |
|---|---|---|
| Helm | repo-server `helm template` | `valueFiles`, inline `values`, `parameters`; not `helm install` — Argo CD applies rendered output |
| Kustomize | `kustomize build` | overlays; `images`/`namePrefix` overrides |
| Directory | raw manifests | `recurse`, `jsonnet` |
| Plugin (CMP) | sidecar | custom tooling |

Multi-source Applications (combine a chart from one repo with values from another) exist — useful
for separating app config from chart, but adds a moving part; use when the separation is real.

## syncPolicy — per environment, not one global setting

```yaml
syncPolicy:
  automated:
    prune: true            # delete resources removed from git
    selfHeal: true         # revert manual cluster drift back to git
    allowEmpty: false      # don't let an empty source delete everything
  syncOptions:
    - CreateNamespace=true
    - ServerSideApply=true         # for big CRDs / apply-annotation-size limits
    - ApplyOutOfSyncOnly=true      # only touch drifted resources (faster, less churn)
    - PrunePropagationPolicy=foreground
    - PruneLast=true               # prune after other resources apply (safer ordering)
    - RespectIgnoreDifferences=true
  retry: {limit: 5, backoff: {duration: 5s, factor: 2, maxDuration: 3m}}
```

Environment stance:
- **dev**: `automated` + `selfHeal` + `prune` — fast, self-correcting.
- **prod**: consider `selfHeal: true` carefully — it silently reverts an emergency manual fix
  (and can fight a mutating webhook forever). Many teams run prod with automated sync but alert
  on drift rather than blind self-heal, or use manual sync with approval. State the tradeoff.
- `prune: true` + a bad path/generator change = mass deletion; pair with `PruneLast`, deletion
  protection, and review the diff (cross-ref `argocd-debug` dangerous-states).

## Sync waves and hooks

- **Sync waves** (`argocd.argoproj.io/sync-wave: "-1"`) order resources: negative waves first
  (CRDs, namespaces), then workloads, then post-steps. A resource in wave N must be healthy
  before N+1 — a stuck early wave blocks everything.
- **Resource hooks** (`argocd.argoproj.io/hook: PreSync|Sync|PostSync|SyncFail`): DB migrations
  as PreSync Jobs, smoke tests as PostSync. Set `hook-delete-policy`
  (`HookSucceeded`/`BeforeHookCreation`) or old hook Jobs collide on immutable fields.
- Hooks vs waves: waves order the desired-state apply; hooks run imperative steps around it.

## Resource tracking

`application.instanceLabelKey` / tracking method (`label` vs `annotation+label`): the annotation
method (`argocd.argoproj.io/tracking-id`) avoids the label-collision problems of the legacy
`app.kubernetes.io/instance` label (which other tools also use). Prefer annotation+label tracking
on new installs; changing it on an existing install re-adopts resources (plan it).

## Health and ignoreDifferences

- Built-in health for core k8s kinds; **custom health via Lua** for CRDs Argo CD doesn't
  understand (`resource.customizations.health.<group>_<kind>`) — without it a healthy operator CR
  shows `Progressing` forever.
- `ignoreDifferences` for fields a controller mutates (HPA replicas, webhook-injected fields) —
  scope narrowly (`jsonPointers`/`jqPathExpressions`); ignoring too much masks real drift
  (cross-ref `argocd-debug` OutOfSync taxonomy). `ServerSideApply` + `RespectIgnoreDifferences`
  together handle most defaulting-noise cleanly.

## Verify (read-only)

```bash
argocd app get <app>
argocd app diff <app>                    # desired vs live, read-only
argocd app manifests <app>               # what would be applied
kubectl get application <app> -n argocd -o yaml | yq '.spec.syncPolicy'
```
