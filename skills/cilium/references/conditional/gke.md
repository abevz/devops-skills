# Cilium on GKE

Self-authored conditional reference. Load when the cluster is Google GKE. The key fact: GKE
Dataplane V2 *is* Cilium — but Google-managed.

## Dataplane V2 (managed Cilium) vs self-managed

| | GKE Dataplane V2 | Self-managed Cilium |
|---|---|---|
| What | Google-operated Cilium dataplane | Your own Cilium install |
| Config access | Limited — GKE exposes NetworkPolicy + DPv2 observability, not raw Cilium | Full Helm/CRD control |
| Advanced features (Egress Gateway, BGP, Cluster Mesh, custom encryption) | ❌ not exposed | ✅ available |
| Cluster requirement | Enable at cluster create (`--enable-dataplane-v2`) | GKE **without** DPv2 |

- If the ask is "network policy + basic flow visibility," **DPv2 is the answer** — don't
  self-install Cilium alongside it. You get CiliumNetworkPolicy-style policy and DPv2 metrics/
  observability managed.
- If the ask needs **Egress Gateway / BGP / Cluster Mesh / custom transparent encryption**, those
  are the reason to self-manage — DPv2 doesn't expose them. That means a GKE cluster **without**
  Dataplane V2, and replacing the default CNI.

## Self-managed caveats (the friction)

- Create the cluster without DPv2 and without GKE network policy
  (`--no-enable-network-policy`) — GKE's own policy enforcement conflicts with Cilium's.
- GKE-managed nodes (COS) support eBPF/modern_ebpf; but node auto-upgrade reimages nodes — the
  DaemonSet re-lands, but plan for churn.
- kube-proxy is GKE-managed; full KPR is constrained on standard GKE — verify what the cluster
  mode allows before designing around `kubeProxyReplacement: true`.
- GKE Autopilot: you don't manage nodes/CNI at all — self-managed Cilium isn't an option; you get
  DPv2. (Autopilot uses gVisor-style isolation on some workloads — relevant to Hubble syscall
  visibility.)
- Google's own guidance steers to DPv2; self-managing is a deliberate choice for the advanced
  feature set, with more operational ownership.

## Decision shortcut

- Need policy/observability only → **DPv2**, done.
- Need Cilium's advanced networking → **GKE Standard without DPv2 + self-managed Cilium**, accept
  the setup friction.
- On Autopilot → **DPv2 only**.
