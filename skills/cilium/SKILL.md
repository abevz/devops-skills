---
name: cilium
description: Use when designing, deploying, or reviewing Cilium — CNI install, kube-proxy replacement, L3/L4/L7 network policies, Hubble observability, transparent encryption, Cluster Mesh, Gateway API/Ingress, Egress Gateway, LB-IPAM/BGP/L2. Mention "cilium", "kube-proxy replacement", "ciliumnetworkpolicy", "hubble", "cluster mesh", "egress gateway", "cilium bgp" as triggers.
license: MIT
compatibility: Works with Claude Code, Codex-style agents, CodeWhale, OpenCode, and other agents that support Agent Skills-style instructions.
---

# Cilium

## When to use

Use when planning, installing, configuring, or reviewing any part of Cilium: the CNI/datapath,
kube-proxy replacement, network policies (including L7), Hubble, encryption, multi-cluster
Cluster Mesh, Gateway API/Ingress, Egress Gateway, or LB-IPAM/BGP/L2 announcements. For
troubleshooting live drops/connectivity ("traffic dropped by policy", stuck endpoints) use the
`cilium-debug` skill instead — this skill is design/deploy/review, that one is symptom triage.

## Goal

Get Cilium's many features configured correctly and safely for the cluster's actual needs —
without turning on complexity (encryption, mesh, BGP) that isn't warranted, and without the
foot-guns each feature carries (datapath mode mismatches, policy default-deny lockouts,
kube-proxy-replacement prerequisites).

## Workflow

1. **Scope the ask to a feature area** and load its reference — don't reason about all of Cilium
   at once:

   | Area | Reference |
   |---|---|
   | Install, datapath (tunnel vs native), IPAM, kube-proxy replacement, CNI migration, managed clusters | `references/install.md` |
   | CiliumNetworkPolicy / ClusterwideNetworkPolicy, L3/L4/L7 (HTTP/DNS/Kafka), toFQDNs, default-deny, deny policies, host firewall | `references/network-policies.md` |
   | Hubble / Relay / UI, flow observability, metrics, export | `references/hubble.md` |
   | Transparent encryption: WireGuard vs IPsec | `references/encryption.md` |
   | Multi-cluster Cluster Mesh, global services | `references/clustermesh.md` |
   | Gateway API, Ingress, Egress Gateway, LB-IPAM, BGP, L2 announcements | `references/gateway-egress.md` |

2. **Confirm the baseline** before feature work: Cilium version, datapath mode, whether
   kube-proxy replacement is on (many features — Egress Gateway, some LB paths — require it),
   kernel version (eBPF features have floors), and the underlying cluster (managed vs self-hosted
   changes IPAM and routing).
3. **Match feature to need, not to novelty** — encryption, Cluster Mesh, BGP each add real
   operational cost; recommend them when the requirement (compliance, multi-region HA, bare-metal
   L2) exists, and say so plainly when it doesn't.
4. **Check prerequisites and blast radius** — many Cilium changes are cluster-wide and disruptive
   (datapath mode change, kube-proxy replacement toggle, encryption enablement all touch every
   node). Plan them like a migration (`migration-plan` skill): staged, reversible, with a
   connectivity check between steps.
5. **GitOps the config** — Helm values + Cilium CRDs live in the GitOps repo; changes go through
   PR and staged rollout, not `cilium` CLI mutations on a live cluster.
6. **Verify with read-only tools** — `cilium status`, `cilium config view`, `hubble observe`,
   `cilium bgp peers`, `cilium clustermesh status` confirm state without changing it.

## References

- `references/install.md`, `references/network-policies.md`, `references/hubble.md`,
  `references/encryption.md`, `references/clustermesh.md`, `references/gateway-egress.md` — one
  per feature area, each with decision tables, Helm/CRD examples, prerequisites, and gotchas.

## Safety rules

- This is a design/deploy-planning/review skill: do not run `cilium install/upgrade`,
  `cilium clustermesh enable`, apply CiliumNetworkPolicies, or toggle datapath/kube-proxy/
  encryption on a live cluster — propose Helm values and manifests for the user to apply.
- Cluster-wide changes (datapath mode, kube-proxy replacement, encryption, CNI migration) can
  sever all pod networking if done wrong — always frame them as staged, reversible rollouts with
  a rollback path, never a single flip.
- A default-deny CiliumNetworkPolicy or a wrong `endpointSelector` can cut off production
  traffic instantly — propose policies in a way that's validated (audit/observe first via
  Hubble) before enforcement.
- Prefer read-only verification (`cilium status`, `hubble observe`, `cilium bgp routes`) in any
  investigation; never restart agents or run `cilium cleanup`-class commands without explicit
  confirmation.

## Output format

```
## Feature area & baseline (version, datapath, kube-proxy replacement, kernel, cluster type)
## Recommendation (what to enable/change, and explicitly what NOT to)
## Prerequisites & blast radius
## Proposed config (Helm values / CRDs, for user review)
## Rollout plan (staged, with per-step verification and rollback)
## Verification (read-only commands)
```

## Quality checklist

- [ ] The relevant feature reference was consulted, not reasoned from memory
- [ ] Prerequisites checked (kube-proxy replacement, kernel floor, datapath compatibility)
- [ ] Cluster-wide/disruptive changes are framed as staged reversible rollouts
- [ ] Complexity (encryption/mesh/BGP) is recommended only against a real requirement
- [ ] Policies are validated (Hubble observe) before enforcement, never blind default-deny
- [ ] Nothing was applied to a live cluster; verification is read-only
