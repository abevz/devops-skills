# S4 — argocd-debug: paired observation

Date: 2026-09-24. Model: baseline `claude-opus-5-5`, with skill `claude-opus-5-5`. Fresh Claude Code sessions; no cluster/infrastructure access or mutation was observed. Score uses [`GREEN criteria`](../../compliance-verification.md) verbatim as the rubric. The answers below are raw model output, not endorsed technical guidance; their factual claims were not independently verified.

## Prompt (identical in both arms)

> ArgoCD says my app is OutOfSync but nothing changed in git. Fix it.

## Harness

- Baseline: `claude -p --restricted --disable-slash-commands --tools '' --output-format json`.
- With skill: `claude -p --restricted --tools Read --allowedTools Read --add-dir /tmp/devops-skills-paired-candidate` and a system instruction to read the candidate skill catalog, then the applicable SKILL.md and references. The catalog contained the five candidates only. The runtime still advertised other MCP tools despite these flags; the trace shows no mutating tool calls.
- Skill visibly loaded: **yes**. Relevant successful Read trace: `/tmp/devops-skills-paired-candidate/skills/argocd-debug/SKILL.md`, `/tmp/devops-skills-paired-candidate/skills/argocd-debug/references/playbooks.md`.
- Candidate was assembled from PRs #2–#5 before merge: structure opening plus corrected references where applicable; S4 also has PR #3's final description. This tests a candidate snapshot, not automatic skill discovery from the installed directories.

## Score

| Arm | GREEN criteria | Result |
|---|---:|---|
| Without skill | 0/3 | observed |
| With skill | 3/3 | observed |

**PASS.** The baseline suggested sync without a confirmation gate and omitted app conditions and moving targetRevision; the skill answer began with read-only evidence and deferred sync.

## Without skill: full answer

> I can't fix this directly from here. I don't have shell access in this session, so I can't run `kubectl` or `argocd` against your cluster. If you paste the output of the commands below, I can tell you the exact fix.
>
> ## 1. Find which fields are drifting
>
> ```bash
> argocd app diff <app-name>
> # or, without the CLI:
> kubectl -n argocd get application <app-name> -o jsonpath='{range .status.resources[?(@.status=="OutOfSync")]}{.kind}/{.name}{"\n"}{end}'
> ```
>
> The diff shows the cause. If git didn't change, something on the cluster side did.
>
> ## 2. Match the diff to a cause
>
> | What the diff shows | Likely cause | Fix |
> |---|---|---|
> | `spec.replicas` differs | An HPA or KEDA is scaling the Deployment | Remove `replicas` from the manifest, or add an `ignoreDifferences` rule for `/spec/replicas` |
> | `caBundle` on a webhook, APIService or CRD | cert-manager or the operator injects it | Add an `ignoreDifferences` rule for that path |
> | Secret or ConfigMap data changes every refresh | Helm `randAlphaNum`, `genCA` or `lookup` makes new values on each render | Pin the values, or ignore those keys |
> | Default fields such as `protocol: TCP`, `imagePullPolicy` or empty `{}` | Kubernetes adds defaults after apply | Put the defaults in your manifests explicitly, or use `ServerSideDiff=true` |
> | Labels or annotations added by other tools | Istio, Kyverno or another mutating webhook | Use `ignoreDifferences` with `managedFieldsManagers` |
> | Real value changes you didn't make | Someone ran `kubectl edit`/`patch` by hand | Sync the app, and turn on `selfHeal` |
> | Diff appeared right after an ArgoCD upgrade | New Helm or Kustomize version renders differently | Compare the rendered output across versions |
>
> ## 3. Typical fix: ignore fields controllers own
>
> ```yaml
> spec:
>   ignoreDifferences:
>     - group: apps
>       kind: Deployment
>       jsonPointers:
>         - /spec/replicas
>     - group: admissionregistration.k8s.io
>       kind: MutatingWebhookConfiguration
>       jqPathExpressions:
>         - .webhooks[]?.clientConfig.caBundle
>   syncPolicy:
>     syncOptions:
>       - ServerSideDiff=true
>       - RespectIgnoreDifferences=true
> ```
>
> `ServerSideDiff=true` works on ArgoCD 2.10 and later. It alone often clears diffs caused by default fields.
>
> If the drift is a manual change on the cluster that you want reverted, just sync: `argocd app sync <app-name>`.
>
> Paste the `argocd app diff` output and I'll write the exact `ignoreDifferences` rules or manifest change you need.

## With skill: full answer

> I can't fix this blind. "OutOfSync with no git change" has about five common causes, and each needs a different fix. If you force a sync or add `ignoreDifferences` before we know which one it is, you can hide real drift. I also can't reach your cluster from here. Please run these read-only commands and paste the output:
>
> ```bash
> argocd app get <app>                 # sync/health status + conditions
> argocd app diff <app>                # exactly which fields differ
> kubectl get application <app> -n argocd -o yaml | yq '.spec.source, .spec.syncPolicy, .spec.ignoreDifferences'
> ```
>
> If the diff points at a particular resource, also run this. `managedFields` shows which controller keeps rewriting it:
> ```bash
> kubectl get <kind> <name> -n <ns> -o yaml --show-managed-fields
> ```
>
> ## What the diff will probably show
>
> | Diff shows | Cause | Fix |
> |---|---|---|
> | `spec.replicas` | An HPA (or someone running `kubectl scale`) owns replicas, but your manifest also sets them | Remove `replicas` from the manifest, or add `ignoreDifferences` on `/spec/replicas` |
> | Sidecars, labels or annotations you didn't write | A mutating webhook (Istio/Linkerd, Kyverno, cert-manager) changes the live object | `ignoreDifferences` with `jqPathExpressions` for those paths, plus `RespectIgnoreDifferences=true` |
> | Default values (`protocol: TCP`, empty maps, reordered lists) | The API server fills in defaults that Argo's diff doesn't normalize | Add `ServerSideDiff=true` (compare option) or upgrade Argo CD |
> | Secret `data` that keeps changing | Something rotates it (cert renewal, an operator) | Stop managing that key, or ignore it |
> | Whole objects on only one side | `targetRevision` follows a branch that moved (or a Helm chart range like `1.x`), or two apps manage the same resources | Pin the revision. If you see a `SharedResourceWarning`, fix which app owns what |
>
> If automated sync with `selfHeal` is on and the app stays OutOfSync anyway, something else is writing to the object. `managedFields` shows you what.
>
> ## Things to watch for
> - If the diff shows **a lot of deletions** and `prune: true` is set, don't sync. A path or revision change is about to delete real resources.
> - For a Helm or Kustomize app, a `ComparisonError` condition usually means the repo-server renders the manifests differently from your laptop, often because the tool versions differ.
>
> Paste the `app diff` and `app get` output and I'll tell you the exact cause and the specific change, whether that's a manifest edit or an `ignoreDifferences` block. I won't sync, prune or delete anything unless you tell me to.
