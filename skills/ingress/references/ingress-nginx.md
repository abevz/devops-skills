# ingress-nginx: operating what remains, safely

## Retirement status (the load-bearing fact)

The Kubernetes project announced ingress-nginx's retirement in November 2025; best-effort
maintenance ended in **March 2026** — no further releases, **including security fixes**. The
planned successor (InGate) never reached production readiness. Consequences for any review:

- An ingress-nginx deployment today is an unpatched attack surface with a history of severe
  CVEs (the 2025 "IngressNightmare" class hit the admission webhook and snippet handling).
- Every ingress-nginx finding gets the same headline recommendation: a dated migration plan to
  a Gateway API implementation (`references/migration.md`). Hardening below is triage for the
  interim, not a destination.

## Interim hardening (while migration runs)

| Check | Why |
|---|---|
| `allow-snippet-annotations: "false"` (default since v1.9) | Snippets inject raw nginx config — config-injection/RCE class; also the #1 migration blocker |
| `--enable-annotation-validation` (default since v1.12) | Rejects malformed annotation values that historically enabled injection |
| Admission webhook not exposed beyond the API server | The webhook was the IngressNightmare entry point; NetworkPolicy it to the API server only |
| Controller not running as cluster-admin-ish | It reads all Secrets by default (TLS); scope with `--watch-namespace` where the layout allows |
| `enable-ssl-passthrough` only if actually used | It bypasses L7 processing and widens the proxy surface |

## Facts that decide reviews

- **Class selection**: `spec.ingressClassName` is the mechanism; the legacy
  `kubernetes.io/ingress.class` annotation is deprecated — mixed use makes two controllers
  fight over one Ingress. One `IngressClass` may set
  `ingressclass.kubernetes.io/is-default-class: "true"`; more than one default breaks creation
  of class-less Ingresses.
- **Source IP**: behind a cloud LB, client IPs require `externalTrafficPolicy: Local` (plus
  losing even node spreading) or PROXY protocol enabled on *both* LB and controller
  (`use-proxy-protocol: "true"`) — enabling it on one side only breaks all traffic instantly.
- **Body size / timeouts**: `proxy-body-size` (default 1m — the classic "413 on upload"),
  `proxy-read-timeout`/`proxy-send-timeout` (default 60s — long-poll/SSE killers).
- **Canary**: `nginx.ingress.kubernetes.io/canary: "true"` + `canary-weight` /
  `canary-by-header` on a *second* Ingress with the same host/path. Only one canary Ingress per
  backend pair; weights are per-request, not per-session (no sticky canary without affinity).
- **Session affinity**: `affinity: cookie` annotations — flag as migration debt; Gateway API
  session persistence is still maturing and this is where migrations discover coupling.
- **Regex/rewrite**: `use-regex` + `rewrite-target` with capture groups — the single most
  common annotation pair; maps to Gateway API `URLRewrite` filters (see migration reference).
- **Wildcard TLS + default backend**: a `*.example.com` secret on the catch-all server and
  `default-backend-service` for no-match traffic; both need explicit Gateway API equivalents
  (listener per wildcard host, explicit catch-all route).

## Review shortcuts

- `kubectl get ingress -A -o json | jq` for: snippet annotations (blockers), deprecated class
  annotation, regex/rewrite use, canary pairs, affinity — this inventory *is* the migration
  scoping document.
- `kubectl -n ingress-nginx get cm ingress-nginx-controller -o yaml` — the global ConfigMap
  holds the risky toggles (snippets, ssl-passthrough, proxy-protocol).
