---
name: kubernetes-debug
description: Use when debugging a Kubernetes issue such as a failing pod, broken service, or unexpected cluster behavior. Mention "kubectl", "pod is crashing", "debug this deployment", "kubernetes issue" as triggers.
license: MIT
compatibility: Works with Claude Code, Codex-style agents, CodeWhale, OpenCode, and other agents that support Agent Skills-style instructions.
---

# Kubernetes Debug

## When to use

Use when a Kubernetes-hosted workload is failing, not starting, not receiving traffic, or
behaving unexpectedly, and the cause needs to be diagnosed.

## Goal

Find the probable cause using read-only inspection, with commands the user can run to confirm,
and a safe fix — without mutating cluster state unless explicitly asked.

## Diagnostic routing

Identify the symptom first, then jump straight to its playbook in `references/playbooks.md`
instead of walking the full checklist:

| Symptom | Playbook section |
|---|---|
| Pod stuck `Pending` | Pod Pending — scheduler event text → cause table |
| `CrashLoopBackOff` | CrashLoopBackOff — exit code/reason → cause table |
| `ImagePullBackOff` / `ErrImagePull` | ImagePull — pull error text → cause table |
| `OOMKilled` restarts | OOMKilled — usage-pattern classification |
| Service unreachable / connection refused | Service chain — 6-link walk (selector → endpoints → readiness → ports → bind address → NetworkPolicy) |
| Name resolution errors / 5s latencies | DNS — CoreDNS, FQDN vs short names, ndots, netpol on port 53 |
| Node `NotReady` | Node — conditions → pressure/kubelet/CNI |

Use the general workflow below when the symptom doesn't match a playbook or spans several.

## Workflow

1. Narrow scope: namespace, workload name, and symptom (CrashLoopBackOff, Pending, no traffic,
   OOMKilled, etc.) — and route via the table above if it matches a playbook.
2. Inspect in this order, read-only:
   - `kubectl get pods` / `describe pod` — status, restart count, events
   - `kubectl get events --sort-by=.lastTimestamp` — recent cluster events in the namespace
   - `kubectl logs` (current and `--previous` if crashed) — actual error output
   - the owning controller (Deployment/StatefulSet/DaemonSet) — replica status, rollout status
   - `Service` + `Endpoints`/`EndpointSlice` — selector match, ready addresses
   - `Ingress`/Gateway API resources — routing and backend health
   - referenced `ConfigMap`/`Secret` — do they exist, are keys present (never print secret values)
   - `ServiceAccount` and RBAC (`Role`/`RoleBinding`) — permission errors in logs/events
   - node status and `describe node` — pressure, taints, allocatable resources, if scheduling
     is stuck
   - probes and resource requests/limits on the pod spec
   - `NetworkPolicy` — if connectivity is denied unexpectedly
3. Correlate evidence to a probable cause before proposing a fix.
4. Propose the safe fix, and the exact commands to verify it worked.

## Safety rules

- Prefer read-only `kubectl get/describe/logs` commands first, always.
- Do not run `kubectl apply`, `delete`, `edit`, `rollout restart`, `scale`, `cordon`, or `drain`
  unless the user explicitly asks for the mutation.
- Never print the contents of a `Secret` value — only confirm existence/keys.
- If a fix requires applying a manifest, show the diff/manifest and the exact command; let the
  user run it, or only run it yourself after explicit confirmation.

## Output format

```
## Observations
<what was seen, from which command>

## Evidence
<relevant log lines / events / describe output, quoted>

## Probable cause
## Commands to verify
## Safe fix
<manifest/command, plus explicit note that it requires user confirmation to apply>
```

## Quality checklist

- [ ] Only read-only commands were run without explicit permission
- [ ] Probable cause is backed by quoted evidence, not guessed
- [ ] No secret values were printed
- [ ] The proposed fix includes a verification step
- [ ] Any mutating command is clearly marked as requiring confirmation
