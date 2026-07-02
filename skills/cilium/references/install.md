# Cilium install, datapath, IPAM, and kube-proxy replacement

Self-authored reference (no third-party source). The foundational setup decisions. Deployment-
planning: propose Helm values, never run `cilium install` against a live cluster.

## Datapath mode — how pod traffic crosses nodes

| Mode | How | Needs | Use when |
|---|---|---|---|
| Tunnel / overlay (VXLAN default `8472/UDP`, or Geneve `6081/UDP`) | Encapsulates pod traffic node-to-node | Nothing from the underlay — works anywhere | Default; underlay you don't control, cross-subnet nodes |
| Native routing | Pod CIDRs routed by the underlay directly, no encap | Underlay/cloud must route pod CIDRs (routes, or cloud integration) | Flat network, performance-sensitive, you control routing |

- Native routing needs `routingMode: native` + `ipv4NativeRoutingCIDR` set, and either
  auto-direct-node-routes (same L2) or a cloud/BGP integration to program routes. Wrong = silent
  cross-node blackhole.
- MTU: tunnel subtracts encap overhead (~50 bytes VXLAN); encryption subtracts more. MTU
  mismatches surface as "large responses hang, small ones work."

## IPAM mode — where pod IPs come from

| Mode | Source | Use when |
|---|---|---|
| Cluster Pool (default) | Cilium allocates from a cluster-wide pool, carves per-node | Default, self-managed |
| Kubernetes | Per-node PodCIDR from the k8s controller-manager | Cluster already assigns PodCIDRs |
| ENI (AWS) | Real VPC IPs via ENIs | EKS wanting pods as first-class VPC citizens |
| Azure / GKE | Cloud-native IPAM | Managed cloud integration |
| Multi-pool / CRD | Named pools by selector | Segmenting IP ranges per tenant/zone |

ENI/cloud IPAM ties pod IP capacity to instance ENI limits — a real constraint on small nodes;
flag it in capacity planning.

## kube-proxy replacement

Cilium replaces iptables-based kube-proxy with eBPF service handling — faster at scale, and a
prerequisite for several features (Egress Gateway, some LB/BGP paths, socket-LB).

```yaml
kubeProxyReplacement: true            # was "strict" in older versions
k8sServiceHost: <API_SERVER_IP>       # REQUIRED when no kube-proxy exists to reach the API
k8sServicePort: 6443
```

- Prereqs: kernel **4.19.57+ / 5.1+** for full replacement; without kube-proxy present, Cilium
  must be told how to reach the API server (`k8sServiceHost/Port`) or the agent can't start —
  the classic "install another CNI then replace kube-proxy" bootstrap trap.
- LB algorithm: `loadBalancer.algorithm: maglev` (consistent hashing, better for DSR/backend
  churn) vs `random`. DSR (`loadBalancer.mode: dsr`) preserves client source IP and avoids the
  reply hairpin, but needs underlay that won't drop asymmetric routing; SNAT is safer default.
- Partial replacement (`kubeProxyReplacement: false` + specific features) exists for coexistence
  with kube-proxy during migration.
- Verify: `cilium status | grep KubeProxyReplacement`, `cilium service list`.

## Install method

- `cilium` CLI (`cilium install`) is fine for labs; **Helm chart `cilium/cilium` is the GitOps/
  production path** — values in git, upgrades reviewed. The CLI wraps Helm anyway.
- Pin the chart version; Cilium minor upgrades occasionally change datapath — read upgrade notes,
  and Cilium's own guidance is one minor at a time, with `cilium upgrade` pre-flight checks.
- Hubble/encryption/etc. are values on the same chart, not separate installs.

## Replacing another CNI (migration)

- The disruptive path: old CNI removed, nodes drained/rebooted, Cilium takes over — pod IPs
  change, connectivity drops during the switch. Plan as a real migration (`migration-plan`).
- Newer Cilium supports **per-node live migration** (dual-CNI via a second CIDR, cordon → migrate
  → uncordon node by node) — far less disruptive; verify the version supports it before promising
  zero-downtime.
- Never a single flip on a running cluster; stage per node pool with connectivity checks.

## Managed clusters — the caveats that bite

| Platform | Note |
|---|---|
| GKE Dataplane V2 | *Is* Cilium, but Google-managed — you don't self-configure it; self-managed Cilium needs GKE without DPv2 + specific flags |
| EKS | Either ENI IPAM (VPC-native) or overlay; can CNI-chain with aws-node or fully replace it — decide, don't run both |
| AKS | "Azure CNI powered by Cilium" (managed) vs BYOCNI self-managed Cilium |
| kubeadm/bare metal | Full control; you own routing/BGP/L2 for LB |

On managed control planes you often can't remove kube-proxy or the cloud CNI cleanly — confirm
what the platform allows before designing around full replacement. For the deep per-cloud
specifics (ENI vs overlay, VPC CNI removal, DPv2, Azure CNI Powered by Cilium, BYOCNI), load the
matching `conditional/{eks,gke,aks}.md` — those override this generic table.

## Verification (read-only)

```bash
cilium status                          # agent/operator health, KPR, datapath, IPAM, encryption
cilium config view                     # effective config (what's actually enabled)
cilium status --verbose | grep -iE "routing|masquerad|kubeproxy|ipam"
cilium connectivity test               # runs a test workload — NOT read-only; confirm with user first
```

`cilium connectivity test` deploys pods and generates traffic — treat as a mutating action
(needs user go-ahead), unlike the `status`/`config view` reads.
