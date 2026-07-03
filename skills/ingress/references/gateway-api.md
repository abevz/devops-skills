# Gateway API: resource model, attachment, TLS, implementations

## Resource model (role-oriented by design)

| Resource | Owned by | Purpose |
|---|---|---|
| `GatewayClass` | Implementation/platform | Which controller implements Gateways of this class |
| `Gateway` | Cluster operator | Listeners: port/protocol/hostname/TLS; where the LB lives |
| `HTTPRoute` / `GRPCRoute` | App teams | Host/path/header matching → backendRefs, filters |
| `TLSRoute` / `TCPRoute` / `UDPRoute` | App teams | Non-HTTP passthrough/L4 (experimental channel) |
| `ReferenceGrant` | Target namespace owner | Allows cross-namespace refs (route→backend, gateway→secret) |

The split is the point: app teams write routes in their namespaces; only the operator touches
listeners/LB config. Reviews should reject designs where app teams write Gateways.

## Route attachment — where "my route does nothing" lives

1. Route's `parentRefs` names the Gateway (and optionally `sectionName` for one listener).
2. The listener's `allowedRoutes.namespaces.from` must admit the route's namespace
   (`Same` is the default — the classic silent failure for routes in app namespaces; use
   `Selector` or `All` deliberately).
3. Listener `hostname` and route `hostnames` must intersect; no intersection = silently not
   attached.
4. Cross-namespace `backendRefs` (and Gateway→TLS-secret refs) need a `ReferenceGrant` in the
   *target* namespace — missing grant surfaces as `ResolvedRefs: False`, reason
   `RefNotPermitted`.

**Status conditions are the diagnostic surface** — always read them before theorizing:
Gateway/listener: `Accepted`, `Programmed`, `attachedRoutes` count; Route (per parent):
`Accepted`, `ResolvedRefs`. `kubectl describe httproute <r>` quoting the condition message is
the equivalent of hubble evidence in `cilium-debug`.

## TLS

- Listener `tls.mode: Terminate` (secret ref, HTTPRoute behind it) vs `Passthrough`
  (TLSRoute, SNI-based, backend holds the cert).
- cert-manager issues Gateway listener certs directly (Gateway resource annotated with the
  issuer; its HTTP-01 solver speaks Gateway API — enable the controller's Gateway API flag).
  Certificate stuck → `cert-manager-debug`.
- Cross-namespace secret refs need ReferenceGrant (above) — centralizing certs in one
  namespace is a valid pattern *with* explicit grants.

## Filters and traffic features (core, portable)

- `RequestRedirect` (HTTP→HTTPS, host redirects), `URLRewrite` (path prefix replace —
  the `rewrite-target` successor), `RequestHeaderModifier`/`ResponseHeaderModifier`,
  `RequestMirror`.
- **Weighted `backendRefs`** replace annotation-based canary: two backends with `weight` on one
  route rule — portable, reviewable, no second Ingress hack.
- Matching: path (Exact/PathPrefix/RegularExpression), headers, query params, method.
- Anything beyond these lives in implementation policy CRDs (BackendTrafficPolicy,
  SecurityPolicy, etc.) — usable, but mark it as vendor-coupled in reviews.

## Choosing an implementation

| If… | Then |
|---|---|
| Cilium is the CNI (kube-proxy replacement on) | Cilium Gateway API support first — no extra data plane to run |
| Istio is already in the cluster | Istio's Gateway API support — same control plane, mesh-integrated |
| No mesh/CNI gateway; want a dedicated, conformance-focused proxy | Envoy Gateway (reference-quality Envoy control plane) |
| Teams want nginx semantics kept | nginx-gateway-fabric (the maintained NGINX path post-ingress-nginx) |
| Already invested in Traefik/HAProxy/kgateway | Their Gateway API support — check conformance reports first |

Check the project's published **conformance report** (Core vs Extended support per feature)
against the feature list you actually need — implementations differ most at the edges
(TLSRoute, session persistence, infrastructure attributes).

## Channels and versions

- **Standard channel** CRDs = stable (Gateway, HTTPRoute, GRPCRoute, ReferenceGrant);
  **experimental channel** adds alpha resources/fields (TCPRoute/UDPRoute/TLSRoute,
  BackendTLSPolicy, session persistence). Don't build production on experimental fields
  without flagging the upgrade risk — experimental CRDs can change shape between releases.
- CRDs install/upgrade separately from the implementation — version-skew between CRDs and
  controller is a real failure mode; pin both in GitOps and upgrade CRDs first per the
  implementation's compatibility matrix.
