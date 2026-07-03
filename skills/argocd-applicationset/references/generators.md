# ApplicationSet generator patterns

Self-authored reference (no third-party source). Generator selection, staged-rollout designs,
and the failure modes each choice buys.

## Generator decision table

| Source of truth for "what deploys where" | Generator | Watch out for |
|---|---|---|
| Small, static env/cluster list | `list` | Stale entries deploy forever; a removed element **deletes the generated app** (and prunes if automated) |
| Clusters registered in ArgoCD (cluster secrets) | `clusters` + label selector | Selector too broad = new cluster gets everything the moment it's registered; label clusters deliberately (`env=prod`, `tier=critical`) |
| Git directory layout (`apps/*/envs/*`) | `git` directories | A misplaced directory instantly becomes an Application; path convention is your admission control |
| Per-file config (`config.json` per app/env) | `git` files | File content becomes template params — schema-validate in CI or a typo ships to the template |
| App list × env list | `matrix` | Cartesian product: N apps × M envs; one template bug hits N×M apps at once |
| Defaults + per-env overrides | `merge` | Merge keys must match exactly or the override silently doesn't apply |
| Preview envs per PR | `pull request` | Requires prune-on-close to work or PRs leak namespaces; cap resources via quota on the preview target |

## Blast-radius control patterns

**Staged env rollout (the default recommendation).** Don't let one commit hit all envs: track
different revisions per stage —

```yaml
generators:
  - list:
      elements:
        - env: dev
          revision: main
        - env: staging
          revision: release        # promoted by merging main → release
        - env: prod
          revision: v1.42.0        # promoted by tagging
template:
  spec:
    source:
      targetRevision: '{{ .revision }}'
```

Promotion = a git operation (merge/tag) that is itself reviewed. The anti-pattern:
`targetRevision: main` for every element — every merge deploys everywhere simultaneously.

**Progressive syncs** (`strategy.rollingSync`, ApplicationSet controller ≥0.4/argocd 2.6+,
gated behind `ApplicationsetProgressiveSyncs` feature flag): sync generated apps in labeled
waves (dev → staging → prod) with `maxUpdate` per step. Verify the flag is actually enabled
before recommending — silently ignored otherwise.

**Deletion protection.** In order of preference:
1. `preserveResourcesOnDeletion: true` in `syncPolicy` (ApplicationSet-level) — deleting the
   ApplicationSet or dropping a generator element orphans resources instead of deleting them.
2. `applicationsSync: create-update` (annotation/policy) — generator changes never delete apps.
3. Controller `--policy create-only` — most conservative, cluster-wide.
Review question: "what happens when someone deletes one line from the list generator?" must
have a known, intended answer.

## Template correctness review points

- Go-template mode (`goTemplate: true`) vs fancy `{{path.basename}}` params — mixed syntax in
  one template silently renders literals. Pick one, check every field.
- Name templates: `'{{ .cluster }}-{{ .app }}'` must stay ≤63 chars and unique across all
  combinations — long cluster names truncate and collide.
- Missing-key behavior: `goTemplateOptions: ["missingkey=error"]` — without it, a typo'd
  parameter renders as `<no value>` inside a namespace or revision field and syncs somewhere
  unintended. This is the single highest-value line in an ApplicationSet spec.
- Values files per env: `helm.valueFiles: [values-{{ .env }}.yaml]` — what happens for an env
  without that file? Helm errors (good) unless `ignoreMissingValueFiles: true` masks it (then a
  new env silently deploys chart defaults — usually not what prod wants).

## AppProject scoping

Every ApplicationSet template should set a non-default `project`, and that AppProject should
pin: `sourceRepos` (this repo only), `destinations` (the namespaces/clusters this set may
touch), `clusterResourceWhitelist` (usually empty — no cluster-scoped resources). This is the
containment when a generator goes wrong: a bad element can only deploy inside the project
fence. `project: default` in an ApplicationSet template = unbounded blast radius; flag it.

## Review one-liners

```bash
# What would this ApplicationSet generate? (read-only dry run)
argocd appset generate <file.yaml> -o json | jq '.[].metadata.name'

# Which apps came from which ApplicationSet
kubectl get applications -n argocd -o json | jq -r '.items[] | select(.metadata.ownerReferences[]?.kind=="ApplicationSet") | .metadata.ownerReferences[0].name + " -> " + .metadata.name'

# Deletion-protection audit
kubectl get appsets -n argocd -o json | jq -r '.items[] | .metadata.name + ": preserveOnDelete=" + (.spec.syncPolicy.preserveResourcesOnDeletion // false | tostring)'
```

## Cross-cluster infrastructure consistency (Cluster Mesh and friends)

When member clusters form a shared fabric — Cilium Cluster Mesh, a stretched service mesh, a
shared CA — per-cluster drift stops being cosmetic and becomes a network incident (one cluster
on an older CNI chart or missing a cross-cluster allow policy drops real traffic). The cluster
generator with label selectors is the tool: label the members (`mesh: prod`), generate one
Application per member from the same chart/policy source, and drift shows up as OutOfSync
instead of as mystery packet loss. The Cilium side of this pairing lives in
`cilium/references/clustermesh.md`.
