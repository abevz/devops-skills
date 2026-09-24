# Upgrade checks: skew, scans, matrix, drains, per-distro notes

## Version skew rules (upstream policy)

| Component | Allowed skew vs kube-apiserver | Consequence |
|---|---|---|
| kube-apiserver (HA peers) | n-1 during rolling upgrade | Upgrade control-plane nodes one at a time |
| kubelet | up to **n-3** older (never newer) | Node pools can lag several hops; a kubelet *ahead* of the API server is invalid — control plane always goes first |
| kube-proxy | not newer than any API server it contacts; up to three minors older than the API server and up to three minors older or newer than its kubelet (two minors for versions before 1.25) | Matching the node's kubelet is an operational preference, not the upstream skew limit |
| kube-controller-manager / scheduler | n-1 vs apiserver | Part of the control-plane hop |
| kubectl | ±1 minor of apiserver | Update operator tooling alongside |

Control plane moves **one minor per hop** — 1.31→1.34 is three hops, each with its own gate.
Node pools may batch hops only within the kubelet skew window, and only *after* the control
plane is at target.

HA API-server skew can narrow the allowed kubelet and kube-proxy versions. Recheck the full
[upstream version-skew policy](https://kubernetes.io/releases/version-skew-policy/) for the
actual old and target versions before scheduling a hop.

## Removed-API scanning (read-only)

```bash
pluto detect-all-in-cluster -o wide --target-versions k8s=v<target>   # live objects
pluto detect-files -d <gitops-repo>/ --target-versions k8s=v<target>  # what git will re-apply
pluto detect-helm -o wide --target-versions k8s=v<target>             # APIs inside released chart manifests
kubent --target-version <target>                                      # second opinion, cluster-wide
```

- Live-only hits with clean git = drift (something was hand-applied) — a `gitops-review`
  finding in its own right.
- Helm-release hits block *future* `helm upgrade` even when the cluster upgrade itself
  succeeds; the fix is a chart version bump (or `mapkubeapis` for orphaned releases), in git.
- Deprecation warnings from the API server (`kubectl get --warnings-as-errors` in CI, or
  apiserver audit annotations) catch what scanners miss between releases.

## Addon compatibility matrix (fill it, don't wave at it)

| Addon | Where the supported range lives | Typical order |
|---|---|---|
| CNI (Cilium/Calico) | Release notes: supported k8s versions per minor | Often must upgrade **before** the k8s hop that drops its range; Cilium: one Cilium minor at a time too |
| CSI drivers | Driver compatibility table | Before or with the hop; a too-old driver breaks attach on new kubelets |
| Ingress/Gateway | Implementation's compatibility matrix + Gateway API CRD channel | CRDs and controller pinned together (see `ingress` skill) |
| cert-manager | Supported k8s range per release | Usually tolerant; check before, not after |
| Operators (DB, observability, service mesh) | Each project's matrix | The long tail — inventory via `kubectl get deploy -A` + Helm releases; the forgotten operator is the classic post-upgrade breakage |

## Drain readiness (both directions)

```bash
kubectl get pdb -A                                   # coverage AND deadlock scan
kubectl get pods -A -o wide --field-selector spec.nodeName=<node>   # what a drain will evict
```

- **Deadlock check**: any PDB where `ALLOWED DISRUPTIONS` is 0 at steady state
  (maxUnavailable: 0, or minAvailable == current replicas, or selector matching a singleton)
  will hang `kubectl drain` forever — fix the PDB or scale before the window, deliberately.
- **Coverage check**: things that must survive rolling drains but have no PDB.
- Singletons/leader-elected pods: a restart is fine but plan *when*; long batch jobs:
  `karpenter.sh/do-not-disrupt` / scheduling around them; local-storage pods
  (`emptyDir`/hostPath data): drain deletes the data — decision, not surprise.
- Surge capacity: rolling node pools needs headroom (quota on cloud, spare hardware in a
  homelab) — verify before, not mid-drain.

## Canary pool pattern

Upgrade one small node pool (or one node) first; run the cluster's real smoke checks against
it (workload schedules, CNI connectivity, CSI attach, DNS) for a soak period before rolling
the rest. On managed platforms this is a separate node group/pool with the new version; on
kubeadm it's "upgrade one worker, uncordon, observe."

## Per-distro / provider notes

- **kubeadm** — `kubeadm upgrade plan` (read-only preview) → `apply` on the first control-plane
  node, `upgrade node` on the rest, then kubelet/kubectl packages per node with drain/uncordon.
  etcd snapshot immediately before each hop (see `cluster-backup`); PKI renewals ride along
  (`kubeadm certs check-expiration`).
- **k3s** — single binary: control plane = restart with the new version (server nodes one at a
  time), agents after; built-in `etcd-snapshot` before. The system-upgrade-controller automates
  the rollout as CRDs — still one minor per hop.
- **Talos** — `talosctl upgrade-k8s` (control plane components) is separate from `talosctl
  upgrade` (node OS image); both are staged per node, API-driven; the same skew and canary
  rules apply.
- **EKS** — control plane upgrade is a managed one-way API call (no downgrade, ever); then
  managed node groups (rolling with maxUnavailable) or Karpenter (new AMIs via drift — budget
  it), then **EKS addons** (vpc-cni, coredns, kube-proxy have per-k8s-version required
  versions — check `aws eks describe-addon-versions`). Deprecated-API "cluster insights" in
  the console are a free second scan.
- **GKE** — release channels auto-upgrade within a window; check channel + maintenance
  windows/exclusions rather than hand-rolling; node auto-upgrade follows the control plane.
  Pin exclusions around business-critical windows instead of fighting the channel.
- **AKS** — cluster upgrade then node-pool upgrades (separate operations, node-image upgrades
  are a third); LTS versions exist for lagging estates; same one-minor rule.

Managed platforms remove the etcd layer from *your* backup gate but not the workload/PV layer —
the `cluster-backup` inventory still applies before every hop.
