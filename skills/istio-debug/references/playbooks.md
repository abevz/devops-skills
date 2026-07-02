# Istio failure playbooks

Route by the strongest signal available: Envoy response flags from sidecar access logs
(`kubectl logs <pod> -c istio-proxy`), then istioctl output. All commands read-only.

## Envoy response flags → cause

| Flag | Meaning | Cause & diagnosis |
|---|---|---|
| `UH` | No healthy upstream | Destination has zero ready endpoints in Envoy's view: `istioctl proxy-config endpoints <client-pod> --cluster "outbound\|<port>\|\|<svc FQDN>"` — empty? Pods not Ready, or subset labels match nothing (DestinationRule subset vs actual pod labels) |
| `UF` | Upstream connection failure | Three classics: (1) mTLS mismatch — see matrix below; (2) app listens on `127.0.0.1` instead of `0.0.0.0` — sidecar can't forward to it; (3) app port not actually listening — check `containerPort` vs real listen port |
| `NR` | No route configured | The request's Host header matches no route: VirtualService `hosts` vs how the client calls it (short name vs FQDN), gateway binding missing (`gateways:` list), or route rule order shadowing (first match wins) |
| `UO` | Upstream overflow | Circuit breaker tripped: DestinationRule `connectionPool` limits too tight for real traffic — `istioctl proxy-config cluster <pod> --fqdn <svc>` shows thresholds; correlate with `upstream_cx_overflow` stats |
| `URX` | Retry/connect limit exceeded | Retries exhausted against a failing upstream — the *underlying* failure is another flag on the individual attempts; don't tune retries first, find what's failing |
| `UT` | Upstream timeout | Response slower than route timeout (default 15s, or VirtualService `timeout`) — is the timeout wrong or the service actually slow? |
| `DC` | Downstream closed | Client hung up (its own timeout) — look at the *caller's* config, not this service |
| `LR` / `UC` | Connection reset | Local/upstream reset mid-stream — app crashes under load, or protocol mismatch (h2 vs http/1.1 in DestinationRule vs app) |
| `RBAC: access_denied` in body | AuthorizationPolicy denies | `istioctl x authz check <pod>` lists evaluated policies; a namespace-wide deny-all with a missing ALLOW for this path/method/principal |

`%RESPONSE_FLAGS%` is in the default access log format; if logs are disabled, enable Telemetry
API access logging for the workload (a config change — propose it, user applies).

## mTLS mismatch matrix

The failing combination is always "one side requires what the other doesn't send":

| PeerAuthentication (server side) | Client sends | Result |
|---|---|---|
| STRICT | mTLS (has sidecar, DR `ISTIO_MUTUAL` or default) | ✅ works |
| STRICT | plaintext (no sidecar, or DR `tls.mode: DISABLE`) | ❌ reset / `UF` — the classic |
| PERMISSIVE | either | ✅ works (transition mode) |
| DISABLE | mTLS forced by DR `ISTIO_MUTUAL` | ❌ TLS to a plaintext port |

Diagnosis: `istioctl x describe pod <server-pod>` prints effective mTLS per port and flags
conflicts directly. Check *both* directions of config: PeerAuthentication (mesh, namespace,
workload — most specific wins) and any DestinationRule `trafficPolicy.tls` overriding client
behavior. Meshexternal services called with mesh assumptions (no sidecar on the far end) need a
ServiceEntry + DR with appropriate TLS mode, not STRICT expectations.

## Sidecar missing or not intercepting

1. Container count: 2/2 (or 3/3 with init)? If 1/1 — injection didn't happen:
   - namespace label `istio-injection=enabled` (or `istio.io/rev=<revision>` — with revisioned
     installs the *old* label silently does nothing)
   - pod-level `sidecar.istio.io/inject: "false"` override
   - pod created *before* the label was added — injection is admission-time only; rollout
     restart required (mutating — user confirms)
2. Sidecar present but traffic bypasses it: check for `hostNetwork: true` (never intercepted),
   or ports excluded via `traffic.sidecar.istio.io/excludeInboundPorts` annotations.
3. Probes failing after injection: kubelet probes bypass mTLS — Istio rewrites them
   (`rewriteAppHTTPProbe`) — if disabled, HTTP probes against a STRICT pod fail.

## Control-plane sync issues

`istioctl proxy-status`:

- `STALE` / `NOT SENT` for a proxy → config never arrived: istiod overloaded or unreachable
  from that pod (`istiod` logs, network path to :15012), or version skew (proxy image much older
  than istiod).
- Everything SYNCED but behavior wrong → the config itself; stop suspecting distribution.
- After any config change during debugging, re-check `proxy-status` before concluding the change
  "didn't work" — propagation is fast but not instant.

## Gateway / ingress path

Request dies at the gateway (client gets 404/connection refused, no service logs at all):

1. `istioctl proxy-config listeners <gw-pod> -n istio-system` — is there a listener on the
   port? No listener → Gateway resource port/TLS block wrong, or gateway selector doesn't match
   the gateway deployment's labels.
2. `istioctl proxy-config routes <gw-pod>` — does the host appear? Missing → VirtualService
   `hosts` doesn't cover the requested Host/SNI, or its `gateways:` list doesn't name this
   Gateway (`<ns>/<name>` form when cross-namespace).
3. TLS: cert secret must live in the gateway's namespace (`credentialName`); SNI mismatch shows
   as handshake failure before any HTTP exists.

## Reading order for "intermittent 503s"

1. Access logs on *both* sidecars for a failing request ID — which hop, which flag?
2. `UH`-dominant → endpoints flapping: readiness instability, HPA churn, rollouts.
3. `UF`-dominant → mTLS matrix or bind address (above).
4. `UO`/`URX` → DestinationRule circuit breaking self-inflicting under bursts — compare
   thresholds to real concurrency before loosening.
5. Mixed flags during deploys only → rollout race: old endpoints drained while routes still
   reference them; check `terminationGracePeriodSeconds` and drain settings.
