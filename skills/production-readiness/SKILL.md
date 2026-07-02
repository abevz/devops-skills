---
name: production-readiness
description: Use when checking whether an app, service, Kubernetes deployment, Helm chart, Terraform module, or GitOps app is ready to run in production. Mention "production readiness", "is this ready to ship", "prod checklist" as triggers.
license: MIT
compatibility: Works with Claude Code, Codex-style agents, CodeWhale, OpenCode, and other agents that support Agent Skills-style instructions.
---

# Production Readiness

## When to use

Use before something goes live, or during a periodic readiness review of an existing service,
Kubernetes workload, Helm chart, Terraform module, or GitOps-managed app.

## Goal

Identify concrete gaps between the current state and what's needed to run safely in production,
ranked by risk — not a generic checklist recital.

## Workflow

1. Identify what kind of artifact this is (app/service, K8s workload, Helm chart, Terraform
   module, GitOps app) and load the relevant lens below.
2. Check **deployment safety**: rollout strategy, readiness/liveness gating, canary or staged
   rollout, and a documented or automated rollback path.
3. Check **observability**: structured logs, key metrics (RED/USE), dashboards, and alerts tied
   to user-facing symptoms, not just infrastructure noise.
4. Check **security**: least-privilege access, secrets handling, network exposure, dependency/
   image scanning status.
5. Check **resourcing**: requests/limits (K8s), autoscaling, and capacity headroom for expected
   load.
6. Check **resilience**: probes, timeouts/retries, graceful shutdown, backup/restore, and — for
   stateful systems — a tested disaster-recovery path.
7. Check **operability**: config/secrets separated from code, runbook or on-call documentation
   exists, and someone other than the author can operate this.
8. Rank every gap found by risk (what breaks, how badly, how likely) — not by how easy it is to
   fix.

## Safety rules

- This is a review skill: do not deploy, apply, or provision anything.
- Do not claim something is "production-ready" without checking it — call out anything you
  couldn't verify (e.g. no access to metrics/dashboards) as unverified.
- Flag missing rollback or DR capability as high risk even if everything else looks good — it is
  the check most often skipped.

## Output format

```
## Scope
<what was reviewed>

## Blocking gaps
- <gap> — risk: <what breaks / who's affected> — fix: <concrete action>

## Should-fix before scale
- ...

## Nice-to-have
- ...

## Not verified
<things that couldn't be checked from available context>
```

## Quality checklist

- [ ] Rollback/DR path was explicitly checked, not assumed
- [ ] Every gap states the concrete failure it enables, not just "missing X"
- [ ] Findings are ranked by risk, not by ease of fix
- [ ] Unverifiable items are listed separately rather than silently skipped
- [ ] No deploy/apply action was taken
