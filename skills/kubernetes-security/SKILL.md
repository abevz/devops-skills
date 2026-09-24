---
name: kubernetes-security
description: Use when reviewing the security posture of a Kubernetes cluster, namespace, or workload. Mention "kubernetes security review", "is this pod secure", "k8s hardening" as triggers.
license: MIT
compatibility: Works with Claude Code, Codex-style agents, CodeWhale, OpenCode, and other agents that support Agent Skills-style instructions.
---

# Kubernetes Security

## TL;DR checklist

- [ ] Review privilege, host access, identity, filesystem, and RBAC exposure.
- [ ] Explain the exploit path for each material gap.
- [ ] Rank findings by reachable risk and propose scoped controls.

## Key read-only checks

- Read workload securityContext, ServiceAccount bindings, namespace labels, and relevant policies.

## Common pitfalls

- Do not report a checklist item without showing its actual risk in context.

## Agent procedure

Follow the [Workflow](#workflow), [Safety rules](#safety-rules), and [Quality checklist](#quality-checklist) below.

## Quick references

- [references/examples.md](references/examples.md)
- [references/hardening.md](references/hardening.md)

Last verified: unverified

## When to use

Use for a focused security audit of Kubernetes workloads, RBAC, or cluster configuration —
distinct from a general manifest-quality review.

## Goal

Find concrete, exploitable security gaps and rank them by real-world risk, not just checklist
compliance.

## Workflow

Check for, and explain the exploit path for each:

1. **Privileged containers** (`privileged: true`, `hostPID`, `hostIPC`) — full or near-full node
   compromise if the container is compromised.
2. **hostPath mounts** — especially write access to sensitive host paths (`/`, `/var/run/docker.sock`,
   `/etc`) — container escape vector.
3. **hostNetwork** — bypasses NetworkPolicy, exposes the pod on the node's network namespace.
4. **runAsRoot / missing runAsNonRoot** — increases blast radius of a container compromise.
5. **Missing seccomp profile** — no syscall filtering; recommend `RuntimeDefault` at minimum.
6. **Missing readOnlyRootFilesystem** — allows tampering with the container filesystem at
   runtime, complicates forensics.
7. **Missing capability drops** (`drop: [ALL]` plus only the specific `add` needed) — excess
   Linux capabilities widen the attack surface.
8. **RBAC wildcards** (`resources: ["*"]`, `verbs: ["*"]`, especially bound broadly) — check who
   can do what, and whether it's the minimum needed.
9. **Default ServiceAccount usage** — implicit token mount; recommend `automountServiceAccountToken:
   false` unless the pod calls the API server, and a dedicated SA otherwise.
10. **`:latest` or floating image tags** — undermines reproducibility and makes compromise harder
    to trace; recommend pinned digests.
11. **Secret exposure** — secrets mounted as env vars (visible via `describe pod`/process env, and
    often leak into logs/crash dumps) versus mounted as files; secrets committed in plaintext in
    manifests/Helm values.
12. **NetworkPolicy gaps** — is there a default-deny policy, or can every pod in the namespace
    talk to every other pod (and possibly other namespaces) by default?

## References

- `references/examples.md` — bad → good security examples with the exploit path each fix closes.
- `references/hardening.md` — PSA labeling with version pinning, NSA/CISA + OWASP K8s Top 10
  mapping, RBAC privilege-escalation vectors, network exposure taxonomy, namespace isolation
  limits, and read-only kubectl/jq audit one-liners.
- `references/tools.md` — optional local scanners (trivy, gitleaks, kube-linter, polaris,
  conftest) that can supplement this manual audit. Never install these automatically; only
  suggest running ones the user confirms are already available.

## Safety rules

- This is a read-only audit skill: do not modify RBAC, SecurityContext, or NetworkPolicy
  resources on a live cluster.
- Never print secret values, even to illustrate an exposure finding — describe the exposure
  without echoing the content.
- Rank findings by actual exploitability (what an attacker could do), not just by "best practice
  says so."

## Output format

```
## Critical (exploitable now)
<finding — exploit path — fix>

## High
## Medium
## Informational / hardening opportunities
```

## Quality checklist

- [ ] Every Critical/High finding states the concrete exploit path
- [ ] Secret values were never printed
- [ ] RBAC was checked for wildcards and broad bindings, not just presence of RBAC
- [ ] Default-deny NetworkPolicy status was explicitly checked
- [ ] No cluster resource was modified
