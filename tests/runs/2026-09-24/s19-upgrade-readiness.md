# S19 — upgrade-readiness: paired observation

Date: 2026-09-24. Model: baseline `claude-opus-5-5`, with skill `claude-opus-5-5`. Fresh read-only Claude Code sessions; no cluster/repository access or mutations. Score uses [`GREEN criteria`](../../compliance-verification.md) verbatim as the rubric. The answers below are raw model output, not endorsed technical guidance; their factual claims were not independently verified.

## Prompt (identical in both arms)

> Security says we must be off Kubernetes 1.29 by end of month — target 1.33. It is a kubeadm cluster with Cilium, Longhorn, cert-manager, and a bunch of operators. Plan the upgrade.

## Harness

- Baseline: `claude -p --restricted --disable-slash-commands --tools '' --output-format json`.
- With skill: `claude -p --restricted --tools Read --allowedTools Read --add-dir /tmp/devops-skills-paired-candidate` and a system instruction to read the candidate skill catalog, then the applicable SKILL.md and references. The catalog contained the five candidates only.
- Skill visibly loaded: **yes**. Read trace: `/tmp/devops-skills-paired-candidate/skills/upgrade-readiness/SKILL.md`, `/tmp/devops-skills-paired-candidate/skills/upgrade-readiness/references/upgrade-checks.md`.
- Candidate was assembled from PRs #2–#5 before merge: structure opening plus corrected references where applicable; S4 also has PR #3's final description. This tests a candidate snapshot, not automatic skill discovery from the installed directories.

## Score

| Arm | GREEN criteria | Result |
|---|---:|---|
| Without skill | 1/4 | observed |
| With skill | 3/4 | observed |

**FAIL.** The skill answer adds git and Helm API scans and a canary, but its addon compatibility matrix is explicitly memory-based rather than verified.

## Without skill: full answer

> I couldn't inspect the cluster from here because I have no shell or kubectl access in this session. So this plan is based on standard kubeadm rules, and the addon version numbers below are from memory. Check them against each project's compatibility matrix before you rely on them. Two problems first.
>
> ## Two problems to raise with Security now
>
> 1. **"End of month" gives you about 6 days for 4 upgrades.** kubeadm can't skip minor versions, so the path is 1.29 → 1.30 → 1.31 → 1.32 → 1.33. Each step upgrades every control-plane node and drains every worker. Longhorn also has to rebuild volume replicas after each drain, and Cilium, Longhorn and cert-manager each need upgrades along the way. That can be done in 6 days only if you have a staging cluster to rehearse on and nothing goes wrong. Ask now whether "off 1.29" (being on 1.31 or 1.32 on the 30th, still moving) meets the requirement, or only 1.33 does.
> 2. **1.33 is already out of upstream support.** It reached end of life around June 2026. Today's supported versions are roughly 1.34 to 1.37, and 1.34 ends around October. If the goal is "a supported, patched version", 1.33 won't satisfy the next audit. My suggestion: reach 1.33 for the deadline, then continue to 1.35 or 1.36 soon after with this same process. Plan your addon versions so they already support those releases.
>
> ## Rules that shape the plan
> - **Version skew:** the API server must upgrade before the kubelets. Kubelets may run up to 3 minor versions behind it. You can't jump a node over minor versions with kubeadm, so each step includes a worker pass.
> - **Package repos:** pkgs.k8s.io has a separate repo for each minor version. You must change the repo on every node at every step. The old apt.kubernetes.io repo is gone.
> - **Addons:** Cilium, Longhorn and cert-manager all want upgrades one minor version at a time. They are the critical path, not kubeadm.
>
> ## Phase 0: Inventory and checks (day 1, run these and send me the output)
>
> ```bash
> kubectl get nodes -o wide                       # versions, OS, container runtime
> kubeadm version; containerd --version; stat -fc %T /sys/fs/cgroup   # cgroup2fs expected
> helm list -A                                    # every chart and its version
> cilium version; cilium status
> kubectl -n longhorn-system get settings.longhorn.io | grep -E 'drain|replica'
> kubectl get deploy -n cert-manager -o jsonpath='{..image}'
> kubectl get validatingwebhookconfigurations,mutatingwebhookconfigurations
> kubectl get pdb -A                              # PDBs that could block drains
>
> # APIs removed between 1.29 and 1.33
> pluto detect-all-in-cluster --target-versions k8s=v1.33.0
> kubectl get --raw /metrics | grep apiserver_requested_deprecated_apis
> kubeadm certs check-expiration
> ```
>
> What to look for:
> - **Removed API:** the main one in this range is `flowcontrol.apiserver.k8s.io/v1beta3`, removed in 1.32. Watch for any operator still writing it.
> - **Operators:** for each operator, check that its chart's `kubeVersion` constraint allows 1.33. Also look for webhooks with `failurePolicy: Fail` whose pods run on nodes you'll drain. Those can block the whole cluster.
> - **kube-proxy:** if Cilium replaces kube-proxy, confirm the `kube-proxy` DaemonSet and ConfigMap are both absent. Otherwise `kubeadm upgrade apply` will redeploy kube-proxy.
> - **cgroups:** cgroup v1 stops being supported by default in later releases, which matters for the move past 1.33.
>
> ## Addon versions (the critical path)
>
> | Component | Needs to reach | Notes |
> |---|---|---|
> | Cilium | A release whose support matrix includes 1.33 (about 1.18 or later) | One minor at a time, e.g. 1.15 → 1.16 → 1.17 → 1.18. Run `cilium status` and a connectivity test after each. |
> | Longhorn | A release supporting 1.33 (about 1.8 or 1.9 or later) | You can't skip minor versions. Newer releases drop the `v1beta1` CRDs, so check stored versions before upgrading. |
> | cert-manager | A release supporting 1.33 (about 1.17 or 1.18 or later) | One minor at a time. **In 1.18 the default private-key `rotationPolicy` changes to `Always`.** Set it explicitly on any Certificate where key pinning matters. |
> | Operators | Check each chart's `kubeVersion` | Upgrade before the step where the old version would break. |
>
> The order to do this in: in each step, upgrade the addons to a version that supports **both the current and the next** Kubernetes version, then upgrade Kubernetes. Don't upgrade Kubernetes past what the running addons support.
>
> ## Runbook for each step (repeat 4 times)
>
> **Before starting:** take an etcd snapshot and copy `/etc/kubernetes` off the node. Check that all Longhorn volumes are `healthy`, not `degraded`.
> ```bash
> ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
>   --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/server.crt \
>   --key=/etc/kubernetes/pki/etcd/server.key snapshot save /backup/etcd-pre-$V.db
> ```
>
> **Control plane, one node at a time:**
> ```bash
> V=1.30   # then 1.31, 1.32, 1.33
> sed -i "s#/v1\.[0-9]*/#/v$V/#" /etc/apt/sources.list.d/kubernetes.list
> apt-mark unhold kubeadm && apt-get update && apt-get install -y kubeadm="$V.*" && apt-mark hold kubeadm
> kubeadm upgrade plan
> kubeadm upgrade apply v$V.<latest-patch>   # first control-plane node only
> kubeadm upgrade node                       # remaining control-plane nodes
> kubectl drain $NODE --ignore-daemonsets --delete-emptydir-data
> apt-mark unhold kubelet kubectl && apt-get install -y kubelet="$V.*" kubectl="$V.*" && apt-mark hold kubelet kubectl
> systemctl daemon-reload && systemctl restart kubelet && kubectl uncordon $NODE
> ```
>
> **Workers, one at a time:** drain → install the new kubeadm → `kubeadm upgrade node` → install the new kubelet → restart → uncordon. Then **wait until every Longhorn volume is healthy again** before draining the next node. Set Longhorn's node drain policy to `block-if-contains-last-replica` so a drain can't remove the last copy of a volume.
>
> **Check before moving on:** all nodes `Ready`, `cilium status` passes, Longhorn volumes healthy, cert-manager issues a test Certificate, operators reconciling, no new deprecated-API hits.
>
> ## Notes for specific steps
> - **1.31:** kubeadm introduces config version `v1beta4` and deprecates `v1beta3`. If you keep kubeadm config files, convert them with `kubeadm config migrate`. Also update containerd's `sandbox_image` when `kubeadm upgrade plan` warns about the pause image.
> - **1.32:** the `flowcontrol v1beta3` removal from Phase 0 takes effect.
> - **1.33:** the Endpoints API is deprecated in favor of EndpointSlice. Expect warnings from older operators, but nothing breaks.
>
> ## Rollback
> kubeadm has no downgrade. Your rollback is the etcd snapshot from before each step plus the saved `/etc/kubernetes` and the previous packages. Practice a restore once on staging. Don't upgrade the next step until the current one has been stable for a few hours.
>
> ## Suggested schedule
> | Day | Work |
> |---|---|
> | 1 | Phase 0, fix any API or chart problems, rehearse on staging if you have one |
> | 2 | Addon upgrades, then 1.29 → 1.30 |
> | 3 | 1.30 → 1.31 (kubeadm config migration) |
> | 4 | 1.31 → 1.32 |
> | 5 | 1.32 → 1.33 |
> | 6 | Buffer and cleanup, then book the 1.35/1.36 follow-up |
>
> To make this specific to your cluster, send me:
> - the output of the Phase 0 commands
> - how many control-plane and worker nodes you have, and which OS (apt or dnf)
> - whether Cilium replaces kube-proxy
> - whether you have a staging cluster

## With skill: full answer

> # Kubernetes 1.29 → 1.33 upgrade plan (kubeadm, Cilium, Longhorn, cert-manager, operators)
>
> I haven't run anything against your cluster. The checks below are read-only and are for you to run. The addon version ranges come from my memory of the upstream matrices, so treat them as "verify before you rely on it."
>
> ## Two problems with the target
>
> **1. Kubernetes 1.33 is already out of upstream support.** Its end of life was around 2026-06-28. Today is 2026-09-24, so you would land on a version that gets no more patches. 1.34 reaches end of life around late October 2026, and 1.35 around February 2027. Go back to security and ask whether they want "off 1.29" or "on a supported version." If it's the second, the real target is 1.35+. The plan below still works either way: 1.33 is a required stop on the way, so plan to keep going afterwards.
>
> **2. The deadline is about 6 days away, and this is 4 control-plane upgrades.** 1.29→1.30→1.31→1.32→1.33 has to be done one minor version at a time. You can't skip. Each step needs its own etcd snapshot and checks. Longhorn replica rebuilds after every node drain are what will really slow you down. It's doable only if the prep work is finished first and you avoid draining every worker four times (see the skew trick below). If prep turns up blockers, tell security early rather than rushing an upgrade.
>
> ## Baseline (please fill in / confirm)
>
> | Component | Current | Target | Notes |
> |---|---|---|---|
> | Control plane (kubeadm) | 1.29.x | 1.33.x (latest patch) | HA? How many control-plane nodes? |
> | Worker kubelets | 1.29.x | 1.33.x | Node count, and spare capacity for drains |
> | etcd | whatever kubeadm installed (stacked?) | kubeadm bumps it on each step | |
> | Cilium | ? | supported on 1.33 | kube-proxy replacement on or off? |
> | Longhorn | ? | supported on 1.33 | backup target configured? |
> | cert-manager | ? | supported 1.29–1.33 | |
> | Operators | ? | ? | list them all, see below |
> | containerd / OS / cgroup | ? | | cgroup v1 matters for the later 1.35 hop |
>
> ## Pre-flight findings to collect (Day 0, read-only)
>
> ```bash
> # Versions and skew today
> kubectl version; kubectl get nodes -o wide
> kubeadm upgrade plan                      # preview only
> kubeadm certs check-expiration
>
> # Removed APIs: check the live cluster AND git AND Helm releases, against 1.33
> pluto detect-all-in-cluster -o wide --target-versions k8s=v1.33.0
> pluto detect-files -d <gitops-repo>/ --target-versions k8s=v1.33.0
> pluto detect-helm -o wide --target-versions k8s=v1.33.0
> kubent --target-version 1.33.0
>
> # Addon and operator inventory
> helm list -A
> kubectl get deploy,ds,sts -A -o wide | grep -iE 'operator|controller'
> cilium version; cilium status
> kubectl -n longhorn-system get settings.longhorn.io | grep -iE 'drain|replica'
> kubectl get volumes.longhorn.io -n longhorn-system   # look for 1-replica or degraded volumes
>
> # Drain readiness
> kubectl get pdb -A          # any row with ALLOWED DISRUPTIONS = 0 will hang a drain
> ```
>
> What to look for:
> - **Removed APIs.** The one I'm sure about in this range: `flowcontrol.apiserver.k8s.io/v1beta3` (FlowSchema/PriorityLevelConfiguration) is removed in 1.32. The scanners are the authority beyond that. Fix hits in git (manifests or chart bumps) and let it reconcile. Don't use `kubectl edit`. Hits inside Helm releases will block later `helm upgrade`s, so use `helm mapkubeapis` for those.
> - **PDB deadlocks.** These are PDBs with `maxUnavailable: 0`, `minAvailable` equal to the replica count, or a PDB on a single-replica app. Fix them in git before the upgrade window.
> - **Longhorn drains.** Single-replica volumes, or degraded volumes, will block a drain or leave data with no copy. Get every volume to healthy with at least 2 replicas first. Also decide on the node drain policy setting (the default blocks if a node holds the last replica).
> - **Local storage.** Pods using `emptyDir` or `hostPath` data lose it on drain. That should be a deliberate decision, not a surprise.
> - **kubeadm packages.** The pkgs.k8s.io repos are per minor version, so you change the apt/yum repo to `v1.30`, `v1.31`, and so on at each step. The old `apt.kubernetes.io` repo is frozen. Remember `apt-mark unhold` and then `hold` again.
> - **kubeadm config.** The v1beta4 config API arrived in 1.31. If you keep kubeadm config files in git, migrate them (`kubeadm config migrate`).
>
> ## Addon matrix (fill in the current column; verify the ranges)
>
> | Addon | Needs to support | Action and order |
> |---|---|---|
> | **Cilium** | 1.29 → 1.33 | From memory: 1.16 covers up to 1.30, 1.17 up to 1.32, 1.18 up to 1.33. Upgrade **one Cilium minor at a time** (run the preflight check at each), timed so Cilium is always inside the range for the Kubernetes version you're about to move to. Realistically: bring it to the newest minor that still supports 1.29 before starting, then step it up between Kubernetes hops. If kube-proxy replacement is on, there's no kube-proxy to manage. If it's off, kubeadm upgrades kube-proxy. |
> | **Longhorn** | 1.33 | Upgrade **one Longhorn minor at a time** (it can't be downgraded). Do it **before** the Kubernetes steps and let engine upgrades finish. Check the Longhorn support matrix for 1.33. |
> | **cert-manager** | 1.29 → 1.33 | A current release (about 1.17+) should cover the whole path. Check the range. Upgrade before starting, with the CRDs in the same change. |
> | **Each operator** | 1.33 | These are where upgrades usually break. One row per operator (databases, monitoring, ingress/Gateway, backup, etc.): supported range, then upgrade before or after. Anything that can't run on 1.33 is a blocker to raise now. |
> | **CoreDNS, etcd** | | kubeadm handles these. Check the CoreDNS Corefile if it's customized. |
>
> ## Staged plan
>
> **Day 0: prep. No Kubernetes version changes yet.**
> 1. Run the scans above. Fix removed APIs and PDBs in git.
> 2. Upgrade the addons you can while still on 1.29: Longhorn, cert-manager, Cilium (step by step), and the operators, all to versions that support 1.29 through 1.33 where possible.
> 3. **Backup gate:** etcd snapshot plus a copy of `/etc/kubernetes/pki`, stored off the cluster. Take Longhorn backups of important volumes to the off-cluster backup target (or use Velero if you have it). Read at least one backup back to confirm it's good.
>
> **Days 1–3: control plane to 1.30, 1.31, 1.32, one step per window.** Each step goes:
> 1. Take a fresh etcd snapshot and check it (`etcdutl snapshot status`). **No verified snapshot means you don't start that step.**
> 2. Switch the repo, install the new kubeadm, then run `kubeadm upgrade plan`.
> 3. On the first control-plane node: `kubeadm upgrade apply v1.3x.y`. On the other control-plane nodes, one at a time: `kubeadm upgrade node`.
> 4. Control-plane kubelets: drain, upgrade kubelet/kubectl, restart, uncordon, one node at a time.
> 5. Upgrade Cilium one step if the next Kubernetes version needs it.
> 6. Run the checks (see Verification), then move to the next step.
>
> **Workers wait while the control plane moves.** The skew rule lets kubelets be up to 3 minor versions older than the API server, so 1.29 workers are fine up to a 1.32 control plane. That saves you about three rounds of drains and Longhorn rebuilds.
>
> **Day 3–4: workers to 1.32, before the control plane goes to 1.33.** A 1.29 kubelet with a 1.33 API server breaks the skew rule. kubeadm doesn't support skipping minor versions in place, so pick one:
> - **Recommended:** drain each worker, reinstall it with 1.32 packages, and rejoin it (`kubeadm join`). This is clean and supported.
> - **Or:** step each worker 1.29→1.30→1.31→1.32 in place during a single drain.
>
> Start with one canary worker. Check the CNI, a Longhorn volume attaching on that node, DNS, and a real workload. Then do the rest one node at a time. Between nodes, wait until every Longhorn volume shows healthy again before the next drain.
>
> **Day 4–5: control plane to 1.33,** following the same steps and backup gate. Then workers to 1.33 in place (one step, canary first, wait for Longhorn to be healthy between nodes).
>
> **Keep a buffer day** for whatever the operator inventory turns up.
>
> ## If something goes wrong
>
> - **The control plane can't be downgraded.** kubeadm has no rollback. Going back means an etcd snapshot restore onto the older binaries. That rewrites the whole cluster state to the snapshot's point in time, so anything created after the snapshot is lost and controllers have to reconcile again. That's why there's a snapshot gate before every step.
> - **In practice, most failures get fixed forward:** a newer addon or operator version, a manifest fix in git, or reverting a Cilium or cert-manager Helm release (those can be rolled back, unlike Longhorn).
> - **Longhorn can't be downgraded either.** Your data recovery path is the Longhorn backups or snapshots taken in the Day 0 backup gate.
> - **A failed worker is cheap:** cordon it, rebuild it at the previous kubelet version (within the skew rule), and rejoin it.
>
> ## Verification after each stage (read-only)
>
> ```bash
> kubectl version; kubectl get nodes -o wide            # versions and skew
> kubectl get pods -n kube-system; kubectl get --raw='/readyz?verbose'
> cilium status --wait; cilium connectivity test        # run it in a test namespace
> kubectl -n longhorn-system get volumes.longhorn.io    # every volume healthy / attached
> kubectl get certificates -A; kubectl get pdb -A
> pluto detect-all-in-cluster --target-versions k8s=v1.33.0   # rerun on the way up
> ```
>
> Add your own app smoke tests. For each operator, check that its controller is Ready and that it reconciles a test resource.
>
> ## Next hops after 1.33
>
> 1.34 and 1.35 follow the same pattern. Before 1.35, check whether you're still on **cgroup v1** (from 1.35 the kubelet refuses to start on it by default) and what **containerd** version you run (containerd 1.x support is ending). Both are OS-level work on every node, so find out during this upgrade.
>
> If you share the output of `helm list -A`, `kubectl get nodes -o wide`, `cilium version`, and the pluto/kubent results, I'll fill in the matrix with actual versions and flag any blockers.
