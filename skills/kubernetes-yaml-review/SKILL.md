---
name: kubernetes-yaml-review
description: Use when reviewing raw or rendered Kubernetes manifests before apply, including Deployments, Services, Helm output, and Kustomize output. Mention "review this manifest", "review this YAML", or "k8s manifest review"; use helm-review for chart templates/values and pr-review for a diff with no manifest focus.
license: MIT
compatibility: Works with Claude Code, Codex-style agents, CodeWhale, OpenCode, and other agents that support Agent Skills-style instructions.
---

# Kubernetes YAML Review

## When to use

Use when reviewing raw or rendered Kubernetes manifests before they're committed or applied —
Deployments, StatefulSets, Services, Ingress, RBAC, etc.
Use `helm-review` instead for chart templates, values, or upgrade behavior and `pr-review`
for a diff without a specific manifest focus.

## Goal

Catch manifest-level mistakes that cause outages, security gaps, or failed upgrades, before they
reach a cluster.

## Workflow

Check each manifest against:

1. **Labels/selectors** — do selectors match template labels exactly? Will a label change
   orphan existing pods or break a rolling update?
2. **Probes** — readiness and liveness probes present, pointing at something that reflects real
   health, with sane `initialDelaySeconds`/`timeoutSeconds` (liveness probes too aggressive can
   cause restart storms).
3. **Resources** — `requests` and `limits` set for CPU/memory; flag missing requests (breaks
   scheduling/QoS) and limits set far above requests without justification.
4. **securityContext** — `runAsNonRoot`, dropped capabilities, `readOnlyRootFilesystem`,
   `allowPrivilegeEscalation: false`, no `privileged: true` unless justified.
5. **serviceAccount/RBAC** — is a dedicated ServiceAccount used, or the `default` one? Does the
   bound Role/ClusterRole grant more than needed (wildcards, cluster-admin)?
6. **PodDisruptionBudget** — present for anything expected to stay available during voluntary
   disruptions (node drains, upgrades)?
7. **HPA** — if autoscaling is expected, is an HPA defined with sane min/max and target metric?
8. **Affinity/anti-affinity** — are replicas spread across nodes/zones, or can they all land on
   one node?
9. **Image tags** — no `:latest` or floating tags in anything meant for production; pinned digest
   or version preferred.
10. **Config/secrets** — are secrets referenced via `Secret`/external-secrets rather than
    hardcoded in the manifest or env var literals?
11. **Namespace assumptions** — does the manifest hardcode a namespace that won't match every
    environment, or rely on `kubectl` context defaults?
12. **Upgrade safety** — rolling update strategy (`maxUnavailable`/`maxSurge`) appropriate for the
    workload; StatefulSet update strategy considered if ordering matters.

## References

- `references/reliability.md` — deep-dive on probes (liveness vs readiness vs startup, the
  cascading-restart rule), graceful shutdown/preStop, rollout strategy arithmetic, QoS classes,
  the CPU-limit throttling question, cross-resource checks (HPA↔PDB), and read-only
  verification one-liners.
- `references/examples.md` — bad → good manifest pairs for the most common findings; use them to
  recognize patterns and to show the user what the fix looks like.
- `references/workload-patterns.md` — choosing the right workload kind, StatefulSet/Job/CronJob/
  DaemonSet-specific review points, StorageClass and PVC pitfalls, and API deprecation/drift
  tables with validation one-liners.
- `references/tools.md` — optional local static-analysis tools (kubeconform, kube-score,
  kube-linter, polaris, pluto, yamllint) that can supplement this manual review. Never install
  these automatically; only suggest running ones the user confirms are already available.

## Safety rules

- This is a review skill: do not run `kubectl apply` or `helm install` on the manifests reviewed.
- Flag missing security defaults even if they're "probably fine in this cluster" — call out the
  gap and let the user decide.
- Do not assume an unreviewed dependency (CRD, admission policy) is absent; say when you can't
  verify something without cluster access.

## Output format

```
## Critical (will break or is unsafe as-is)
## Major (works but risky)
## Minor / style
## Not applicable / already good
```

## Quality checklist

- [ ] Selector/label match was checked, not assumed
- [ ] securityContext and RBAC were checked on every workload, not just the first
- [ ] Image tags were checked for `:latest` or floating tags
- [ ] PDB/HPA presence was checked against the workload's actual availability needs
- [ ] No manifest was applied to a cluster
