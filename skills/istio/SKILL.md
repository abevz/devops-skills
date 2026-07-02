---
name: istio
description: Use when installing, configuring, or operating Istio — install/revisions/upgrades, sidecar vs ambient data plane, traffic management (VirtualService/DestinationRule/Gateway), mTLS and AuthorizationPolicy, and telemetry. Mention "istio install", "istio ambient", "ztunnel", "waypoint", "virtualservice", "peerauthentication", "authorizationpolicy", "istio canary upgrade" as triggers.
license: MIT
compatibility: Works with Claude Code, Codex-style agents, CodeWhale, OpenCode, and other agents that support Agent Skills-style instructions.
---

# Istio

## When to use

Use when standing up or operating an Istio mesh: install method and revision-based upgrades,
choosing sidecar vs ambient data plane, traffic management (routing/canary/resilience), security
(mTLS, authorization, JWT, identity/certs), and telemetry. For troubleshooting a live mesh
problem (503 taxonomy, mTLS handshake failures, sidecar not injecting, route not taking effect)
use `istio-debug`. This skill is the design/deploy/configure view.

## Goal

A mesh sized and secured for the actual need — right data-plane mode, mTLS enforced without
locking out legacy clients, authorization default-deny where it matters, upgrades done by
revision (not in place), and telemetry that's useful without exploding cardinality.

## Workflow

1. **Scope to an area** and load its reference:

   | Area | Reference |
   |---|---|
   | Install, profiles, revision/canary upgrades, injection, Istio CNI, gateways | `references/install-upgrade.md` |
   | Sidecar vs ambient (ztunnel/waypoint), migration, when to use which | `references/sidecar-vs-ambient.md` |
   | VirtualService/DestinationRule/Gateway/ServiceEntry, canary, resilience (retries/timeouts/outlier), fault injection | `references/traffic-management.md` |
   | mTLS/PeerAuthentication, AuthorizationPolicy, RequestAuthentication (JWT), SPIFFE identity, CA/certs | `references/security.md` |
   | Telemetry API: metrics, access logs, tracing, Kiali, cardinality control | `references/telemetry.md` |

2. **Confirm the baseline** — Istio version, install method (istioctl/Helm), whether revisions/
   tags are in use, sidecar or ambient (or mixed during migration), mesh-wide mTLS mode, and how
   ingress is done (istio gateway vs Gateway API).
3. **Right-size the mesh** — not every cluster needs full L7 everywhere. Ambient's L4-only default
   (mTLS + identity) with waypoints only where L7 is needed is often the leaner answer; don't
   turn on features (fault injection, complex routing) without a driver.
4. **Security is the point** — mTLS + AuthorizationPolicy are why most people adopt a mesh; a
   mesh running PERMISSIVE mTLS forever with no authz policies is mostly overhead. Move to STRICT
   deliberately (PERMISSIVE → observe → STRICT), default-deny authz where it matters.
5. **Upgrade by revision, never in place** — canary upgrades (new revision alongside, migrate
   namespaces, roll back by relabel) are the safe path; in-place control-plane upgrades risk the
   whole mesh at once.
6. **GitOps the config** — Istio CRDs (VirtualService, DestinationRule, PeerAuthentication, etc.)
   and IstioOperator/Helm values are declarative and belong in git; changes staged and validated
   (`istioctl analyze`) pre-merge.
7. **Verify read-only** — `istioctl analyze`, `istioctl proxy-status`, `istioctl proxy-config`,
   `istioctl x describe`; never apply mesh config or restart istiod as "investigation."

## References

- `references/install-upgrade.md`, `references/sidecar-vs-ambient.md`,
  `references/traffic-management.md`, `references/security.md`, `references/telemetry.md` — one
  per area, with decision tables, CRD/config examples, prerequisites, and gotchas.

## Safety rules

- Design/operate-planning skill: do not apply Istio CRDs, install/upgrade the control plane,
  toggle mTLS mode, or restart istiod/gateways on a live mesh — propose config for the user.
- mTLS and AuthorizationPolicy changes can sever every consumer of a service at once
  (STRICT with a plaintext client; a default-deny with a missing ALLOW) — frame as staged,
  validated (analyze + PERMISSIVE/observe first) rollouts, never a single flip.
- Never propose disabling mTLS mesh-wide as a shortcut; narrow to the affected workload pair.
- Certificate/CA material and JWT signing config are sensitive — reference, never inline or echo.

## Output format

```
## Area & baseline (version, install method, revisions?, sidecar/ambient, mTLS mode, ingress)
## Recommendation (what to configure/change, and explicitly what NOT to over-build)
## Prerequisites & blast radius
## Proposed config (CRDs / IstioOperator / Helm values — for review, istioctl analyze clean)
## Rollout plan (staged, revision-based, with rollback)
## Verification (read-only istioctl checks)
```

## Quality checklist

- [ ] The relevant area reference was consulted, not reasoned from memory
- [ ] Data-plane mode (sidecar/ambient) matches the actual L4-vs-L7 need, not defaulted
- [ ] mTLS moves to STRICT via PERMISSIVE/observe, not a blind flip; authz default-deny scoped
- [ ] Upgrades are revision-based/canary with a relabel rollback, not in-place
- [ ] Config validated with `istioctl analyze`; cardinality considered for telemetry
- [ ] Nothing was applied/upgraded/restarted on a live mesh
