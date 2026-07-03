# Ingress → Gateway API migration

## Phase 0 — inventory (this decides the timeline)

`kubectl get ingress -A -o json` and classify every object:

| Bucket | Contents | Effort |
|---|---|---|
| Mechanical | host/path routing, TLS termination, ssl-redirect, rewrite-target, canary weights | Tool-assisted, low |
| Mappable with thought | affinity/sticky sessions, auth annotations, rate limits, body-size/timeouts | Per-implementation policy CRDs — needs a target-feature check |
| **Blockers** | `configuration-snippet` / `server-snippet` (raw nginx config) | No mechanical translation — each one is re-engineered or dropped; count them first |

`ingress2gateway` (kubernetes-sigs) converts Ingress objects (with provider flags for common
annotation dialects) into Gateway API manifests — treat its output as a **starting point for
review**, never as apply-ready: it can't know your listener layout, allowedRoutes policy, or
ReferenceGrant strategy.

## Annotation → Gateway API mapping

| ingress-nginx | Gateway API |
|---|---|
| `rewrite-target` (+ `use-regex` capture groups) | `URLRewrite` filter, `ReplacePrefixMatch`/`ReplaceFullPath` — capture-group regex rewrites need re-thinking, often into multiple explicit rules |
| `ssl-redirect` / `force-ssl-redirect` | HTTP listener whose route is a single `RequestRedirect` (scheme https, 301) |
| `canary` + `canary-weight` | One route, two weighted `backendRefs` — delete the second-Ingress hack |
| `canary-by-header` | Header match rule routing to the canary backend |
| `proxy-body-size`, timeouts | Implementation policy CRD (e.g. BackendTrafficPolicy) — vendor-coupled, document it |
| `auth-url` / basic-auth | Implementation SecurityPolicy/extension or an explicit auth proxy in the chain |
| `affinity: cookie` | Session persistence — experimental/implementation-specific; verify the target supports it *before* committing the host |
| `whitelist-source-range` | Implementation policy or CiliumNetworkPolicy/NetworkPolicy at the LB/service layer |
| default backend | Explicit catch-all route (lowest-precedence PathPrefix `/`) on the listener |

## Phased cutover (per-host, both stacks live)

1. **Stand up the Gateway stack beside ingress-nginx** — own controller, own LB/IP/hostname.
   Nothing moves yet; conformance-check the features from the inventory against it.
2. **Shadow a low-risk host** — replicate its routing as HTTPRoute, test against the new LB
   endpoint directly (curl/Host header or a test DNS name), compare responses.
3. **Cut DNS for that host** to the new LB with a short TTL. Rollback = revert the DNS record —
   which is why the old Ingress object stays in git, untouched, until the host is confirmed.
4. **Repeat host-by-host**, mechanical bucket first, blockers last (each blocker host carries
   its own re-engineering PR).
5. **Decommission** ingress-nginx only when `kubectl get ingress -A` is empty (or every
   remaining object is class-switched dead config) — then remove the controller, its webhook,
   RBAC, and LB.

## Gotchas that bite mid-migration

- **Wildcard hosts**: one `*.example.com` Ingress may need a wildcard listener + per-team
  routes; decide the listener layout before moving the first wildcard host.
- **TLS secret namespaces**: ingress-nginx read secrets in the Ingress's namespace; a
  centralized Gateway needs ReferenceGrants (or cert-manager re-issuing per-Gateway) — plan
  cert placement, don't copy secrets around.
- **externalTrafficPolicy / PROXY protocol / client-IP** behavior differs per implementation —
  re-verify source-IP-dependent hosts (allowlists, geo, rate limits) explicitly after cutover.
- **Long-lived connections** (websockets, SSE, GRPC streams) — DNS cutover doesn't move
  established connections; drain the old side rather than killing it same-day.
- **Two controllers fighting**: while both run, ensure neither claims the other's objects
  (distinct IngressClass/GatewayClass; no default-class overlap) — the symptom is routing that
  flaps depending on which controller reconciled last.
