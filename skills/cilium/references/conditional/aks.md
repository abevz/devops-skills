# Cilium on AKS

Self-authored conditional reference. Load when the cluster is Azure AKS. Managed Cilium dataplane
vs bring-your-own.

## The two paths

| | Azure CNI Powered by Cilium | BYOCNI (self-managed Cilium) |
|---|---|---|
| What | Azure-managed: Azure CNI handles IPAM, Cilium is the dataplane/policy | You install Cilium yourself |
| Config access | Limited to what AKS exposes | Full Helm/CRD control |
| IPAM | Azure CNI (VNet IPs) — overlay or VNet-integrated | Your choice (cluster-pool overlay, etc.) |
| Advanced features (Cluster Mesh, Egress Gateway, BGP) | ❌ not exposed | ✅ available |
| Setup | Cluster flag at create | `--network-plugin none`, then install Cilium |

- **Azure CNI Powered by Cilium** is the managed answer for policy + eBPF dataplane without
  running Cilium yourself — enable at cluster create; you get CiliumNetworkPolicy enforcement and
  Azure-integrated IPAM.
  - IPAM sub-choice: **overlay** (Cilium/Azure overlay CIDR, escapes VNet IP limits) vs
    **VNet/node-subnet** (pods get VNet IPs, routable, but bounded by subnet size). Pick overlay
    unless pods must be VNet-routable.
- **BYOCNI** (`--network-plugin none` at create) is required for the advanced feature set
  (Cluster Mesh, Egress Gateway, BGP, custom encryption) — you own the full install and upgrades.

## Caveats

- You can't convert an Azure-CNI-Powered-by-Cilium cluster into a BYOCNI self-managed one in
  place — it's a create-time decision. Choosing wrong means a cluster rebuild/migration; decide
  the feature needs up front.
- kube-proxy: managed in the powered-by mode; BYOCNI lets you do full KPR.
- Node images (Ubuntu/Azure Linux) have modern kernels — modern_ebpf driver is fine.
- Windows node pools: Cilium is Linux-dataplane; Windows nodes fall back to Azure CNI behavior —
  don't assume policy parity on Windows workloads.

## Decision shortcut

- Policy + managed eBPF dataplane → **Azure CNI Powered by Cilium** (overlay IPAM default).
- Cluster Mesh / Egress Gateway / BGP → **BYOCNI**, decided at cluster creation.
