---
name: istio-debug
description: Use when debugging Istio service mesh issues such as sidecar injection failures, mTLS errors, 503s, or VirtualService routing problems. Mention "istio", "envoy sidecar", "mtls error", "virtualservice not routing" as triggers.
license: MIT
compatibility: Works with Claude Code, Codex-style agents, CodeWhale, OpenCode, and other agents that support Agent Skills-style instructions.
---

# Istio Debug

## When to use

Use when traffic in an Istio mesh misbehaves — 503s between services, mTLS handshake failures,
VirtualService routes not taking effect, sidecars missing or unready — and the cause needs to be
found. Also covers reviewing Istio config (VirtualService/DestinationRule/Gateway) before apply.

## Goal

Locate the failing hop (app → local sidecar → remote sidecar → app) and the config that breaks
it, backed by istioctl/Envoy evidence, and propose a fix — without mutating live mesh config.

## Workflow

1. **Run the analyzer first** — `istioctl analyze -n <ns>` catches a large share of config
   mistakes outright (missing DestinationRule subsets, gateway/host mismatches, deprecated
   fields). Cheap, read-only, always first.
2. **Sidecar presence** — is the sidecar actually there? `kubectl get pod -o jsonpath` for
   container count, namespace label `istio-injection=enabled` (or revision label), pod-level
   `sidecar.istio.io/inject` overrides. Pods created *before* injection was enabled never get one
   — check pod age vs. label age.
3. **Control-plane sync** — `istioctl proxy-status`: every proxy should be `SYNCED`. A proxy
   stuck `STALE`/`NOT SENT` means the config you're staring at never reached Envoy — debug
   istiod connectivity/version skew before debugging routing rules.
4. **mTLS mismatch** — for connection resets / `upstream connect error ... TLS error`:
   - `PeerAuthentication` (namespace and mesh-wide) vs. `DestinationRule.trafficPolicy.tls` —
     the classic break is STRICT PeerAuthentication with a DestinationRule forcing `DISABLE`, or
     a mesh-external service addressed as if it were in the mesh
   - `istioctl x describe pod <pod>` summarizes effective mTLS per port
   - plaintext legacy clients hitting a STRICT namespace show as raw TCP resets on port 15006
5. **Routing rules** — for "VirtualService ignored":
   - `host` must match how the client actually calls it (short name resolves within the
     namespace; FQDN elsewhere)
   - subset routes require a `DestinationRule` defining that subset, and pods must carry the
     subset's labels — `istioctl analyze` flags missing ones, `istioctl proxy-config routes
     <pod>` shows what Envoy actually has
   - rule ordering: first match wins; a catch-all route above a specific one shadows it
6. **503 taxonomy** — read the Envoy response flags in sidecar access logs (`kubectl logs <pod>
   -c istio-proxy`): `UH` = no healthy upstream (endpoints empty/unready), `UF` = upstream
   connect failure (mTLS or the app not listening), `NR` = no route (routing config),
   `UO`/`URX` = circuit breaker / retry budget exhausted (check DestinationRule
   `outlierDetection`/`connectionPool` — overly tight settings self-inflict 503s under load).
7. **Gateway path** — for ingress issues: Gateway `hosts`/`port`/TLS config vs. VirtualService
   `hosts` + `gateways` binding, and whether the gateway pod's Envoy actually got a listener
   (`istioctl proxy-config listeners <gw-pod> -n istio-system`).
8. **Correlate and conclude** — name the failing hop and the exact config object; propose the
   minimal change and the command that verifies it.

## Safety rules

- Read-only first, always: `istioctl analyze/proxy-status/proxy-config/x describe`,
  `kubectl get/describe/logs`. None of these mutate state.
- Do not apply, edit, or delete Istio resources (VirtualService, DestinationRule,
  PeerAuthentication, Gateway) on a live cluster — propose the manifest; the user applies it.
  A bad mTLS or routing change can cut off every consumer of a service at once.
- Never suggest disabling mTLS mesh-wide as a diagnostic shortcut — narrow to the affected
  workload pair with evidence instead.
- Do not restart istiod or inject/eject sidecars without explicit confirmation.

## Output format

```
## Observations
<analyzer output, proxy-status, sidecar presence>

## Evidence
<envoy response flags / istioctl output, quoted>

## Failing hop
<app → local sidecar → remote sidecar → app: which link, and why>

## Root cause
## Fix
<config diff, marked as requiring user confirmation to apply>

## Verification
```

## Quality checklist

- [ ] `istioctl analyze` was run (or its unavailability noted) before manual digging
- [ ] Proxy sync status was checked before debugging routing config
- [ ] 503s were classified by Envoy response flags, not guessed at
- [ ] mTLS was checked from both sides (PeerAuthentication and DestinationRule)
- [ ] Only read-only commands were run; no mesh config was applied
