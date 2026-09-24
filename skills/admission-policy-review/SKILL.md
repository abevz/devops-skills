---
name: admission-policy-review
description: Use when authoring, reviewing, or rolling out Kubernetes admission enforcement with Kyverno, OPA Gatekeeper, or Pod Security Admission. Mention "kyverno policy", "gatekeeper constraint", or "admission control"; use kubernetes-security for broad posture audits and runtime-security-review for detection after admission.
license: MIT
compatibility: Works with Claude Code, Codex-style agents, CodeWhale, OpenCode, and other agents that support Agent Skills-style instructions.
---

# Admission Policy Review

## When to use

Use when **deploying** an admission engine (Kyverno install, HA, webhook failure policy),
designing or reviewing cluster admission policy: choosing between PSA/Kyverno/Gatekeeper,
authoring policies, planning an audit→enforce rollout, or debugging "why did admission block
this." This is the enforcement counterpart to `kubernetes-security` (which
finds the gaps policies should close) and `supply-chain-security` (whose signatures policies
verify).
Use `kubernetes-security` instead for a general posture audit and `runtime-security-review`
instead for Falco/Tetragon/Tracee detection after admission.

## Goal

A policy set that actually blocks the dangerous stuff, rolled out without breaking running
teams, with exceptions that are visible, justified, and expiring — not a policy engine
installed and left in audit mode forever.

## Workflow

1. **PSA first, engine second** — Pod Security Admission (built-in, free) covers the
   privileged/hostNamespace/capabilities baseline via namespace labels. A policy engine adds
   what PSA can't: image rules, required fields (resources/probes), signature verification,
   mutation, exceptions. Review question: is the engine duplicating PSA (waste) or extending it?
2. **Engine choice** — Kyverno: YAML-native policies, built-in `verifyImages`, mutation,
   generation; the default for teams without Rego skills. Gatekeeper: Rego/CEL, better for
   complex cross-resource logic and orgs already invested in OPA. Both wrong answers: two
   engines at once, or CEL ValidatingAdmissionPolicy ignored when the need is one simple rule
   (it's built-in since 1.30, no engine required).
3. **The minimal deny set** — check coverage of: privileged, hostNetwork/hostPID/hostIPC,
   hostPath (beyond an explicit allowlist), missing runAsNonRoot, missing resources,
   `:latest`/tag-only images, disallowed registries, unsigned images (where supply chain
   exists). Anything beyond this set needs a named incident or requirement justifying it.
4. **Audit → enforce rollout** — new policies land in `Audit`/`Warn`, violations get a burn-down
   period with owners notified, then `Enforce` — per-policy, not big-bang. Review the PolicyReport/
   audit results *before* recommending enforce: enforcing with 40 open violations = 40 broken
   deploys.
5. **Scope discipline** — system namespaces (kube-system, CNI, CSI, monitoring agents) need
   deliberate exclusions, not wildcards: exclude by namespace for infra, never exclude entire
   policies cluster-wide because one DaemonSet complained.
6. **Exceptions with expiry** — every exception (Kyverno PolicyException / Gatekeeper
   excludedNamespaces) carries: reason, owner, expiry date in an annotation. An exception file
   without dates is where policy goes to die.
7. **Test before ship** — `kyverno test` (with test manifests in the policy repo) or `gator
   test` for Gatekeeper run in CI against good/bad fixture manifests; a policy change is code —
   it goes through PR like everything else (GitOps-managed policies, not kubectl-applied).
8. **Watch the failure mode** — `failurePolicy: Fail` on the webhook means "webhook down =
   nothing schedules" (secure, fragile); `Ignore` means "webhook down = policies silently off"
   (available, bypassable). Know which is set and whether that's the intended tradeoff; check
   webhook replica count/PDB if `Fail`.

## References

- `references/kyverno-patterns.md` — working policy patterns (require digest, verifyImages
  with identity pinning, deny privileged, required resources/probes, mutate-to-defaults),
  PolicyException with expiry, audit-mode reporting queries, and rollout sequencing.
- `references/kyverno-deployment.md` — installing/operating Kyverno itself: controller
  components, the HA-vs-failurePolicy tradeoff decided at install, never-gate-system-namespaces,
  webhook scoping/timeouts, CRD upgrade discipline, GitOps integration (PolicyReport diff noise,
  mutate-vs-drift, sync-wave bootstrap ordering), and deployment troubleshooting.

## Safety rules

- This is a review/design skill: do not apply, modify, or delete policies, webhooks, or
  namespace PSA labels on a live cluster — propose manifests for the user.
- Never recommend jumping straight to Enforce on a cluster with running workloads — audit
  results first, always.
- When a policy blocks something in production, the answer is a scoped, expiring exception or a
  workload fix — not disabling the policy cluster-wide; say so explicitly.

## Output format

```
## Current coverage vs minimal deny set
<table: rule — PSA/engine/missing — mode (audit/enforce)>

## Rollout risks
<what enforcing would break today, from audit data or manifest review>

## Proposed policies / changes
<manifests, each with audit-first rollout note>

## Exception hygiene findings
<exceptions without reason/owner/expiry>
```

## Quality checklist

- [ ] PSA baseline was considered before engine policies
- [ ] Every proposed policy has an audit-mode step before enforce
- [ ] System-namespace exclusions are scoped, not wildcarded
- [ ] Exceptions carry reason, owner, and expiry
- [ ] Webhook failurePolicy tradeoff was stated
- [ ] No policy was applied to a live cluster
