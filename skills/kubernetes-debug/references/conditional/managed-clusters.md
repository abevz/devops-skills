# Debugging on managed clusters (EKS / GKE / AKS)

Self-authored conditional reference. Load when the cluster is a managed cloud one — what's
different (and often hidden) when debugging vs a self-hosted cluster. All read-only.

## What you can't see (control plane is managed)

- **No direct control-plane access**: kube-apiserver/scheduler/controller-manager/etcd logs
  aren't `kubectl logs`-able. Use the cloud's log export instead: EKS control plane logs →
  CloudWatch (must be *enabled* per log type; often off by default), GKE → Cloud Logging, AKS →
  diagnostic settings to Log Analytics. "The scheduler is misbehaving" needs the cloud console,
  not kubectl.
- Audit logs likewise live in the cloud logging backend, not the cluster.

## Nodes get auto-remediated (hides root cause)

- Managed node groups **auto-repair/auto-replace** unhealthy nodes. A node that was
  NotReady/pressured may be *gone* by the time you look — the symptom (evicted/rescheduled pods)
  outlives the cause. Check the cloud's node-group/autoscaler events and instance history, not
  just `kubectl get nodes` now.
- Auto-upgrade reimages nodes on the provider's schedule — a "it broke overnight with no deploy"
  can be a node image bump. Check node age/image vs the incident time.

## Networking specifics

| | EKS | GKE | AKS |
|---|---|---|---|
| Default CNI/IPAM | VPC CNI (ENI, VPC IPs; ENI limits cap pods/node) | DPv2 (Cilium) or kubenet | Azure CNI / overlay |
| Pod-density limit | instance ENI/IP limit → `ImagePull`-like `FailedCreatePodSandBox`/no-IP | subnet size | subnet/overlay |
| LoadBalancer | AWS LB Controller (NLB/ALB) provisioning delay | GCP LB | Azure LB |

- "Pod stuck ContainerCreating, `failed to assign an IP`" on EKS = **ENI/IP exhaustion** on the
  node (VPC CNI), not a generic CNI bug — check per-instance IP limits / prefix delegation. This
  is the most common managed-EKS-only pod-pending cause and isn't in the generic Pending playbook.
- LoadBalancer `EXTERNAL-IP <pending>` for minutes is often normal cloud LB provisioning, or an
  IAM/subnet-tag misconfig (LB controller can't provision) — check the controller's logs, not the
  Service.

## Identity / access failures

- Pods calling cloud APIs use IRSA (EKS) / Workload Identity (GKE) / Workload Identity (AKS).
  `AccessDenied` from a pod is usually a missing/misscoped role annotation on the ServiceAccount,
  not app code — check the SA annotation and the trust policy (cross-ref `secrets-management`).
- IMDS access from pods may be blocked (hop limit / IMDSv2) — an app expecting node-role
  credentials via IMDS fails; the fix is IRSA/WI, not opening IMDS.

## Practical stance

- Before deep kubectl archaeology on a managed cluster, check: (1) is control-plane/audit logging
  even enabled in the cloud, (2) did a node get auto-replaced/upgraded around the incident,
  (3) is this a cloud-IPAM/LB/IAM issue masquerading as a k8s issue. These three account for most
  "only happens on managed" confusion.
