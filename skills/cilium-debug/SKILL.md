---
name: cilium-debug
description: Use when debugging Cilium CNI networking issues such as dropped traffic, CiliumNetworkPolicy problems, or connectivity failures in a Cilium-based cluster. Mention "cilium", "hubble", "traffic dropped by policy", "cilium connectivity" as triggers.
license: MIT
compatibility: Works with Claude Code, Codex-style agents, CodeWhale, OpenCode, and other agents that support Agent Skills-style instructions.
---

# Cilium Debug

## When to use

Use when pod-to-pod, pod-to-service, or egress traffic misbehaves in a Cilium cluster — drops,
timeouts, DNS failures, policy that blocks more or less than intended — and the cause needs to
be found. Also covers reviewing CiliumNetworkPolicy before it's applied.

## Goal

Pinpoint where and why Cilium is dropping or misrouting traffic, backed by Hubble/agent
evidence, and propose a fix — without mutating live policies or cluster state.

## Workflow

1. **Baseline health first** — `cilium status` (via CLI or `kubectl exec` into a cilium agent
   pod), controller failures, unreachable nodes, and `kubectl get pods -n kube-system -l
   k8s-app=cilium` for crashlooping agents. Half of "network is broken" is one unhealthy agent.
2. **Endpoint state** — `cilium endpoint list` on the node hosting the affected pod: is the
   endpoint `ready`? Which identity did it get? An endpoint stuck in `waiting-for-identity` or
   `regenerating` explains total connectivity loss.
3. **Watch the traffic** — `hubble observe --from-pod <ns>/<pod> --to-pod <ns>/<pod>` (or
   `--verdict DROPPED`): the verdict and drop reason (`Policy denied`, `DNS`, `CT: connection
   tracking`) usually names the culprit directly. This is the single highest-value command in a
   Cilium investigation.
4. **Policy evaluation** — if verdict is `Policy denied`:
   - list `CiliumNetworkPolicy`/`CiliumClusterwideNetworkPolicy` *and* plain `NetworkPolicy`
     selecting either endpoint — they combine, and a forgotten default-deny in one of the three
     is the classic cause
   - remember direction: traffic needs egress allowed on the source AND ingress allowed on the
     destination
   - check identity labels the policy matches against (`cilium identity get <id>`) — a selector
     matching a label the pod doesn't actually carry silently matches nothing
5. **DNS-aware policy** — for `toFQDNs` rules: DNS itself must be allowed (port 53 to kube-dns)
   *before* the FQDN rule can ever learn IPs; `hubble observe --type l7 --protocol dns` shows
   whether DNS queries are seen and answered.
6. **Service path** — for ClusterIP issues with kube-proxy replacement enabled:
   `cilium service list` on the node — does the service have backends? Missing backends with
   healthy pods points at endpoint propagation, not policy.
7. **Cross-node path** — for node-to-node failures: encapsulation/native routing mismatch, node
   firewall blocking VXLAN/Geneve ports or WireGuard (51871/UDP) if encryption is enabled;
   `cilium-health status` shows the node-to-node reachability matrix.
8. **Correlate and conclude** — tie the drop reason to the specific policy/agent/route cause
   before proposing a fix; propose the minimal policy change (or agent remediation) and how to
   verify it with `hubble observe` after applying.

## References

`references/playbooks.md` — drop-reason → cause → fix tables (policy denial walkthrough,
kube-proxy-replacement service issues, node-to-node/encryption, DNS flows). Route there as soon
as `hubble observe` names the drop reason.

## Safety rules

- Read-only first, always: `cilium status/endpoint list/service list/identity get`,
  `hubble observe`, `kubectl get/describe`. None of these mutate state.
- Do not apply, edit, or delete network policies on a live cluster — propose the manifest and
  the verification command; the user applies it. A wrong policy change can sever production
  traffic instantly.
- Do not restart cilium agents or run `cilium cleanup`-class commands without explicit user
  confirmation — agent restarts briefly disrupt traffic on the node.
- Never suggest "delete the policy" as a diagnostic shortcut in production — narrow the policy
  hypothesis with `hubble observe` evidence instead.

## Output format

```
## Observations
<agent/endpoint health, from which command>

## Evidence
<hubble verdicts / drop reasons / endpoint state, quoted>

## Root cause
## Fix
<policy diff or remediation, marked as requiring user confirmation to apply>

## Verification
<the hubble observe / connectivity check that proves it's fixed>
```

## Quality checklist

- [ ] `hubble observe` (or equivalent drop evidence) backs the conclusion, not policy reading alone
- [ ] Both CiliumNetworkPolicy AND plain NetworkPolicy were checked, both directions
- [ ] DNS allowance was checked before blaming toFQDNs rules
- [ ] Only read-only commands were run without explicit confirmation
- [ ] The fix includes a post-apply verification command
