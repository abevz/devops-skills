# Cilium on EKS

Self-authored conditional reference. Load when the cluster is Amazon EKS. What changes vs a
self-managed install. Deployment-planning: propose config, don't apply.

## The two integration modes (decide first)

| Mode | IPAM | aws-node (VPC CNI) | Pod IPs |
|---|---|---|---|
| Full replacement, overlay | Cilium cluster-pool, VXLAN | **removed** | Cilium CIDR (not VPC) |
| Full replacement, ENI | Cilium **ENI IPAM** (real VPC IPs) | **removed** | VPC IPs via ENIs Cilium manages |
| CNI chaining | aws-vpc-cni for IPAM, Cilium chained for policy/Hubble | **kept** | VPC IPs from aws-vpc-cni |

- ❌ running aws-node (VPC CNI) AND full-replacement Cilium together — pick one. Full replacement
  deletes the `aws-node` DaemonSet; chaining keeps it.
- ENI mode makes pods first-class VPC citizens (routable, SG-addressable at node level) but ties
  pod density to **instance ENI/IP limits** — the same constraint as VPC CNI. Prefix delegation
  raises the ceiling; small instances still cap low. Flag in capacity planning.
- Overlay mode escapes ENI limits (Cilium CIDR) but pods aren't VPC-native (NAT for egress,
  not directly routable from VPC).

## kube-proxy replacement on EKS

- EKS ships the `kube-proxy` managed addon. For KPR: remove/disable it and set
  `k8sServiceHost`/`k8sServicePort` to the EKS API endpoint (no kube-proxy to reach the API).
- Managed node groups reboot on updates — modern_ebpf works on AL2/AL2023 kernels (≥5.8), so the
  driver isn't the blocker; the addon removal + API-endpoint config is.

## Security-group-per-pod caveat

- SG-per-pod is an **aws-vpc-cni** feature; Cilium ENI mode does **not** provide it. If you rely
  on per-pod security groups, either stay on chaining (keep VPC CNI) or replace that control with
  `CiliumNetworkPolicy`/`toEntities`/`toCIDR` (identity-based, arguably better) — decide
  deliberately, don't silently lose SG-per-pod on migration.

## Practical notes

- Load balancing: AWS LB Controller for `Service type=LoadBalancer`/Ingress still applies; Cilium
  handles east-west + policy, ALB/NLB handle north-south (or Cilium Gateway + NLB).
- IRSA for any Cilium AWS API calls (ENI mode) — the operator needs an IAM role
  (`ec2:*NetworkInterface*`, `ec2:AssignPrivateIpAddresses`); scope it, use IRSA not node role
  where possible.
- Migration from VPC CNI: not in-place trivial — pods re-IP; plan per node group (see the CNI
  migration section of `install.md`).
