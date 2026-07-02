# Istio install, revisions, and upgrades

Self-authored reference (no third-party source). Install methods, profiles, revision-based canary
upgrades, injection, CNI, gateways. Deployment-planning: propose config, never install/upgrade a
live mesh.

## Install method

| Method | Use |
|---|---|
| `istioctl install` (+ IstioOperator YAML) | Simple, imperative or config-driven; good for labs and small setups |
| Helm charts (`base`, `istiod`, `gateway`, `cni`, `ztunnel`) | **GitOps/production** — charts in git, ArgoCD/Flux-managed, per-component control |
| Istio Operator (in-cluster) | Deprecated/removed in newer versions — don't build on it |

Profiles (`--set profile=`): `default` (prod baseline), `demo` (everything, high resource — not
prod), `minimal` (control plane only), `ambient`, `empty`/`preview`. Start from `default` or
`ambient` and layer an IstioOperator/values overlay; don't ship `demo` to prod.

## Revisions and canary upgrades (the safe upgrade model)

In-place control-plane upgrades risk the whole mesh at once. Instead run **revisions**:

```bash
# install a new revision alongside the old
istioctl install --revision=1-24-0 -f values.yaml       # (planning: propose, don't run)
# namespaces opt in by revision label, not the generic label
kubectl label ns payments istio.io/rev=1-24-0 --overwrite
# roll workloads to re-inject the new sidecar, validate, then remove the old revision
```

- Revision label `istio.io/rev=<rev>` on the namespace pins which control plane injects; the
  generic `istio-injection=enabled` maps to the *default* revision only.
- **Revision tags** (`istioctl tag set default --revision=1-24-0`) decouple namespaces from
  revision names — namespaces point at a tag (`istio.io/rev=default`), you move the tag to the
  new revision, roll workloads, rollback by moving the tag back. Prefer tags — namespaces don't
  need relabeling per upgrade.
- Data-plane upgrade requires a **workload restart** to pick up the new sidecar (rollout) — plan
  it; the control plane running new while sidecars run old is supported within skew but bounded
  (~2 minor versions).
- Always `istioctl x precheck` / `istioctl analyze` before, and canary one namespace before the
  fleet.

## Sidecar injection

- Admission-webhook, at pod creation: namespace `istio-injection=enabled` (default rev) or
  `istio.io/rev=<rev>` (revisioned) — **not both** (the revisioned one wins/conflicts).
- Pod-level override `sidecar.istio.io/inject: "false"`.
- Pods created **before** the label existed have no sidecar — injection is create-time; roll the
  workload to inject (the #1 "no sidecar" cause, cross-ref `istio-debug`).

## Istio CNI plugin

- Default injection uses an init container needing `NET_ADMIN`/`NET_RAW` to set up iptables —
  which **conflicts with restricted Pod Security Admission**. The **Istio CNI plugin** moves that
  setup to a node plugin, so app pods don't need those capabilities.
- Enable the CNI plugin (`cni` chart) when running PSA `restricted` or when you don't want
  privileged init containers — effectively required for hardened clusters. Ambient uses it too.

## Gateways

- Ingress/egress gateways are **separate Envoy deployments** (not the control plane) — scale and
  secure them independently; the ingress gateway is your edge, size it accordingly.
- Two config styles: Istio's own `Gateway` + `VirtualService`, or **Gateway API**
  (`gateway.networking.k8s.io`) which Istio implements — Gateway API is the forward-looking
  choice; new meshes should prefer it.
- Don't run the gateway in the same trust/scaling profile as random app pods — it terminates edge
  TLS and faces the internet.

## Verify (read-only)

```bash
istioctl version
istioctl analyze -A                     # config problems across the mesh
istioctl proxy-status                   # every proxy SYNCED? revision skew?
kubectl get ns -L istio.io/rev -L istio-injection    # who's on which revision
```
