---
name: ingress
description: Use when designing, deploying, reviewing, or migrating north-south traffic in Kubernetes — ingress-nginx configuration and its retirement, Gateway API (GatewayClass/Gateway/HTTPRoute), choosing an implementation, and Ingress→Gateway API migration. Mention "ingress", "ingress-nginx", "gateway api", "httproute", "expose a service", "ingress migration" as triggers.
license: MIT
compatibility: Works with Claude Code, Codex-style agents, CodeWhale, OpenCode, and other agents that support Agent Skills-style instructions.
---

# Ingress / Gateway API

## When to use

Use when getting traffic *into* a cluster: configuring or reviewing ingress-nginx, designing
Gateway API resources (GatewayClass/Gateway/HTTPRoute and friends), choosing an implementation,
or planning the Ingress→Gateway API migration that ingress-nginx's retirement forces. For the
TLS certificate on the listener failing, use `cert-manager-debug`; for Cilium's or Istio's own
gateway implementation details, their umbrella skills carry the depth — this skill owns the
controller-neutral north-south layer.

## Goal

North-south traffic that is deliberately designed: an implementation chosen against actual
requirements (not habit), routes and TLS expressed portably where possible, no
retired-controller exposure, and any migration done host-by-host with a DNS-level rollback —
never a big-bang cutover.

## Workflow

1. **Scope to an area** and load its reference:

   | Area | Reference |
   |---|---|
   | ingress-nginx: config, annotations, hardening, retirement status | `references/ingress-nginx.md` |
   | Gateway API: resource model, route attachment, TLS, implementations | `references/gateway-api.md` |
   | Ingress→Gateway API migration: mapping, tooling, phased cutover | `references/migration.md` |

2. **Confirm the baseline** — which controller(s) run today and their versions, how many
   Ingress/Route objects and owning teams, how TLS is terminated (cert-manager? passthrough?),
   what sits in front (cloud LB, MetalLB, hostPort), and whether any config relies on
   controller-specific annotations or snippets — those are the migration long poles.
3. **Treat ingress-nginx as end-of-life** — it was retired in March 2026 (no further releases,
   including security fixes). Running it is now an unpatched-attack-surface decision that needs
   an explicit migration timeline, not a default. New deployments should start on Gateway API.
4. **Choose the implementation against requirements** — already running Cilium or Istio usually
   means using *their* Gateway API support before adding a new data plane; otherwise compare on
   conformance level and the specific features needed (see the decision table in
   `references/gateway-api.md`). Don't run more controllers than the cluster needs.
5. **Prefer portable expression** — core Gateway API fields over implementation-specific
   annotations/policies wherever both exist; every vendor-specific escape hatch is future
   migration debt (ingress-nginx snippets are the cautionary tale).
6. **Plan migrations host-by-host** — both stacks run side by side on separate endpoints; hosts
   move one at a time by DNS with a tested rollback; snippet-dependent hosts are inventoried
   first because they have no mechanical translation (see `references/migration.md`).
7. **GitOps the config** — Gateway/HTTPRoute/Ingress objects are declarative and belong in git;
   validate before merge; never hand-edit live routing.
8. **Verify read-only** — `kubectl get gateway,httproute -A`, `kubectl describe` for
   status conditions (`Accepted`, `Programmed`, `ResolvedRefs`), controller logs, and
   `curl`-level checks against the endpoints.

## References

- `references/ingress-nginx.md` — annotation/hardening facts, retirement guidance.
- `references/gateway-api.md` — resource model, attachment/ReferenceGrant, TLS modes,
  implementation decision table, status conditions.
- `references/migration.md` — annotation→field mapping, ingress2gateway, phased plan, gotchas.

## Safety rules

- Design/review skill: do not apply Gateway/HTTPRoute/Ingress changes, switch ingress classes,
  or delete controllers on a live cluster — propose manifests; the user applies them.
- Routing changes sever live traffic instantly when wrong — any change to an existing host's
  routing is framed with a verification step and a rollback (previous manifest or DNS).
- Never propose a big-bang controller cutover; migration is per-host with both stacks running.
- Snippet annotations execute raw controller config — flag any existing use as both a security
  finding (arbitrary config injection) and a migration blocker; never add new ones.
- TLS secrets and any credentials in annotations are secrets — reference a secrets mechanism,
  never inline values.

## Output format

```
## Area & baseline (controller(s), versions, #hosts/objects, TLS story, LB in front)
## Recommendation (what to configure/migrate, and explicitly what NOT to over-build)
## Prerequisites & blast radius
## Proposed config (Gateway/HTTPRoute/values — for review)
## Rollout plan (per-host, side-by-side, DNS rollback)
## Verification (read-only status conditions + endpoint checks)
```

## Quality checklist

- [ ] The relevant area reference was consulted, not reasoned from memory
- [ ] ingress-nginx exposure is called out with a migration timeline, not silently accepted
- [ ] Implementation chosen against requirements (existing mesh/CNI gateway support first)
- [ ] Portable core fields preferred over vendor annotations; snippet use flagged
- [ ] Migration is host-by-host with side-by-side stacks and DNS rollback
- [ ] Nothing was applied to live routing; verification is read-only
