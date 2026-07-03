# Node autoscaling: cluster-autoscaler vs Karpenter

Both act on the same signal — **Pending pods that fit no current node** — and both size purely
from **requests**. Dishonest requests produce over/under-provisioned nodes no tuning can fix.

## Decision table

| Dimension | cluster-autoscaler (CA) | Karpenter |
|---|---|---|
| Model | Grows/shrinks *pre-defined node groups* (ASGs/MIGs/VMSS) | Provisions *right-sized nodes just-in-time* from a `NodePool` spec, no fixed groups |
| Instance choice | Whatever the node group template says | Chooses per pending-pod shape from allowed types (great for mixed/spot/GPU) |
| Bin-packing / cost | Coarse — group granularity | Consolidation actively replaces underutilized nodes with cheaper/fewer |
| Provider support | Everywhere (cloud-provider plugins) | AWS first-class; Azure (AKS node autoprovisioning); others maturing |
| Operational surface | ConfigMap status, per-group flags | CRDs: `NodePool` + provider class (e.g. `EC2NodeClass`), NodeClaims |
| When it's the answer | Simple/uniform groups, provider without Karpenter support | AWS/AKS with diverse workloads, spot, cost pressure |

Managed add-on autoscaling (GKE Autopilot, EKS Auto Mode) bundles this layer entirely — check
before designing anything by hand.

## Karpenter design points

- **NodePool** sets requirements (instance families/sizes, capacity type spot/on-demand,
  zones), limits (max CPU/mem the pool may create), and `disruption`:
  `consolidationPolicy: WhenEmptyOrUnderutilized` (active repacking) vs `WhenEmpty` (timid),
  plus `budgets` to cap how many nodes churn at once (e.g. `10%`, or `0` during business
  hours) — budgets are the guardrail that makes consolidation production-safe.
- **Interruption handling**: spot interruption/rebalance events drain ahead of reclaim — enable
  the interruption queue on AWS; spot without it is an availability roulette.
- **Protection**: `karpenter.sh/do-not-disrupt: "true"` on pods that must not be consolidated
  away (long jobs, leader singletons); PDBs are respected — but a too-strict PDB silently
  *blocks* consolidation and scale-down forever (the classic "why is this empty node still
  here").
- **Drift**: Karpenter replaces nodes whose spec drifted from the NodePool/NodeClass (AMI
  updates roll through automatically) — that's a feature and a change-management surface;
  budget it.

## cluster-autoscaler design points

- Scale-down is utilization-threshold based (default: node under 50% for 10min and all pods
  evictable). Things that silently pin nodes: pods without a controller, local storage,
  restrictive PDBs, `cluster-autoscaler.kubernetes.io/safe-to-evict: "false"` (and the same
  annotation set to `"true"` is how you *allow* eviction for local-storage pods).
- Multiple node groups + expander choice (`least-waste`, `priority`) is the only bin-packing
  lever; heterogeneous needs = many groups = the maintenance Karpenter removes.
- Status lives in the `cluster-autoscaler-status` ConfigMap (kube-system) — scale-up failures
  (quota, out-of-stock instance types) appear there and as Pending-pod events.

## Failure modes (both)

| Symptom | Cause |
|---|---|
| Pods Pending, no node added | Pod doesn't fit any allowed instance/group shape (huge requests, taints/affinity unsatisfiable, NodePool limits hit, cloud quota) — the Pending events + autoscaler status name it |
| Nodes added but pods still Pending | Something else unschedulable: taints without tolerations, topology constraints, volume zone affinity (PV in another AZ) |
| Empty nodes never removed | PDB blocks a pod on them, un-evictable pod (see pins above), or consolidation budget/policy too timid |
| Node churn / workloads restarted "randomly" | Consolidation too aggressive with no budgets; missing do-not-disrupt on intolerant pods |
| Cost went *up* after Karpenter | Requests dishonest (nodes sized to fiction) or consolidation disabled — check both before blaming the tool |

## Fixed hardware (homelab / bare metal)

No provider = no node autoscaling. The equivalent discipline: workload-layer autoscaling within
a **capacity budget** — sum of max replicas × requests must fit the hardware; alert on
allocatable saturation (see `observability-review`) instead of pretending nodes will appear.
