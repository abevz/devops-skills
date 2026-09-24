# Istio sidecar vs ambient data plane

Self-authored reference (no third-party source). The central Istio architecture decision.
Deployment-planning: propose config, don't migrate a live mesh.

## The two modes

| | Sidecar | Ambient |
|---|---|---|
| Data plane | Envoy injected per pod | Node `ztunnel` (L4) + optional per-namespace/service `waypoint` (L7) |
| L4 (mTLS, identity, TCP authz) | via the sidecar | **ztunnel** (DaemonSet), no per-pod proxy |
| L7 (HTTP routing, L7 authz, retries) | via the sidecar | **waypoint** proxy, deployed only where L7 is needed |
| Enroll a workload | inject sidecar → **restart pod** | label namespace/pod → **no restart** |
| Per-pod overhead | one Envoy per pod (CPU/mem) | none for L4; shared waypoint for L7 |
| Maturity | mature, every feature | GA (Istio 1.24+); most features, some edges still catching up |

## How ambient works (the mental model)

- **ztunnel** (per node, DaemonSet): handles mTLS, SPIFFE identity, and L4 authorization for all
  ambient pods on that node, tunneling traffic over **HBONE** (HTTP/2 CONNECT + mTLS). This gives
  you zero-trust mTLS + identity **without any per-pod proxy**.
- **waypoint** (per namespace or per service account): a standalone Envoy you deploy **only when
  you need L7** — HTTP routing, L7 AuthorizationPolicy, retries/faults. Traffic is routed through
  it by the ztunnel. No waypoint = L4-only mesh, which is enough for many "just give me mTLS +
  identity" cases.

## Choosing

- **Ambient** when: you want mTLS/identity across the fleet cheaply (no per-pod Envoy), no-restart
  enrollment matters, L7 is needed only on some services (pay for waypoints there only), or
  per-pod sidecar overhead is a real cost at scale.
- **Sidecar** when: you need a feature not yet solid in ambient, per-pod Envoy tuning
  (EnvoyFilter customizations), or you're on a mature sidecar deployment with no pressing reason
  to move.
- **Mixed** is supported and is the migration path — sidecar and ambient workloads interoperate
  (both speak mTLS); migrate namespace by namespace.

## Migration (sidecar → ambient)

1. Prereq: Istio CNI plugin + ztunnel installed (`ambient` profile / charts).
2. Remove the sidecar (drop the injection label) and add the ambient label on the namespace →
   pods rejoin as ambient **without a restart** for L4 (though rolling to drop the old sidecar
   is cleaner).
3. Add a **waypoint** for namespaces/services that had L7 policy or routing (`istioctl waypoint
   apply`) — without it, L7 AuthorizationPolicy/VirtualService L7 rules won't be enforced (they
   need an L7 proxy in path). A waypoint label alone does not guarantee traversal: traffic may
   bypass it when the waypoint is unavailable or the traffic type is not handled. When L7
   authorization must be enforced, pair the waypoint's L7 policy with an L4
   `AuthorizationPolicy` on the destination workloads that allows only the waypoint's service
   account identity, so bypass traffic is denied by ztunnel.
   [Istio waypoint enforcement](https://istio.io/latest/docs/ambient/usage/waypoint/).
4. Validate mTLS and policy with `istioctl analyze` / Hubble-equivalent flow checks per stage.

## Gotchas

- L7 features (HTTP match/route, L7 authz, retries, fault injection) require a **waypoint** in
  ambient — a VirtualService with HTTP rules does nothing for an ambient service with no waypoint.
- `AuthorizationPolicy` at L4 (principals/ports) works via ztunnel; L7 conditions (paths,
  methods, headers) need the waypoint.
- Resource model shifts: no per-pod sidecar requests/limits, but ztunnel (per node) and waypoints
  (per ns/service) need sizing.

## Verify (read-only)

```bash
istioctl proxy-status                    # ztunnel + waypoint + sidecar proxies
kubectl get pods -n istio-system -l app=ztunnel
istioctl waypoint list -A                # which namespaces/services have L7 waypoints
```
