# Design Notes

## Why these skills

The list was chosen to cover one person's actual daily loop — Kubernetes, ArgoCD/GitOps,
Terraform/OpenTofu, and Go — plus the cross-cutting work that surrounds any of that: writing
commit messages, reviewing PRs, investigating incidents, and communicating clearly in English.
Each skill maps to a distinct, recurring task rather than a broad domain, following the
"curated over comprehensive" philosophy: a compact set of high-quality, narrow skills covering
the bulk of daily work beats 50+ overlapping ones nobody fully trusts or remembers exists.

The initial 20 skills were extended with a second batch of 7 closing identified gaps:
`dockerfile-review` and `cicd-review` (the two supply-chain-facing artifacts the original set
didn't cover), `runbook-writer` and `alert-rule-review` (closing the loop that
`incident-analysis` and `observability-review` open — both those skills demand runbooks and good
alerts, so something has to produce them), `grafana-dashboards` (dashboards-as-code for the same
observability loop), and `cilium-debug`/`istio-debug` (stack-specific network/mesh debugging,
same read-only-first shape as `kubernetes-debug` but with the tool-specific evidence sources —
Hubble verdicts, Envoy response flags — that generic advice misses).

Three of the reviewed third-party repositories were genuinely strong reference material —
`kubernetes-skill` (failure-mode-first content, pure markdown) and `terraform-skill`
(diagnose-first workflow, Response Contract discipline) most of all, with `devops-ai-skill`
contributing the plan-only migration methodology and the optional-tooling idea. None of their
files were copied; every skill here was written from scratch in this repository's own shape, so
it stays small, consistent, and fully auditable in one sitting. See `docs/third-party-review.md`
for the full per-repository breakdown.

## Skills intentionally not created

- **No day-2 operational skills that mutate infrastructure** (namespace provisioning, node
  drains, secret rotation, cluster upgrades). `cluster-skills` has scripts for this, but they
  came with unsafe defaults (no dry-run, no confirmation) and a broken dependency — exactly the
  category this repository avoids by design. If day-2 automation is ever wanted, it should be a
  deliberate, carefully-guarded addition, not a default inclusion.
- **No separate "kubernetes-operations"/"cluster lifecycle" skill** — `production-readiness` and
  `kubernetes-debug` already cover the review and diagnostic angles a markdown-only skill can
  responsibly own; actual node/etcd/upgrade operations need real tooling and cluster access, not
  an agent skill.
- **No generic "devops-assistant" or "kubernetes-swarm-orchestrator" mega-skill** —
  `cluster-skills`' agent-persona layer (Atlas/Shield/Flow/Pulse/etc.) was interesting but out of
  scope: it's a multi-agent orchestration product, not a single reusable skill, and duplicates
  what the 20 skills here already cover individually.
- **No install/setup skill** — deliberately excluded per the project's own safety rules; nothing
  in this repository should ever be the thing that decides to `brew install` or `apt-get install`
  something on the user's machine.
- **No CI/CD pipeline *generator* skill** — `devops-ai-skill`'s `cicd-enhancer` guide was
  reviewed but judged too broad/opinionated about a specific pipeline stack to generalize well.
  The narrower `cicd-review` skill (security/reliability review of existing pipeline definitions)
  was added later instead — reviewing what exists generalizes; generating a pipeline doesn't.

## How overlap was reduced

- `pr-review` (any language, structural/logic review) is kept distinct from `go-code-review`
  (Go-specific idiom/concurrency/context review) and `go-testing-review` (test quality
  specifically) — each has a narrow, non-overlapping "when to use."
- `kubernetes-debug` (live cluster, read-only diagnosis) is kept distinct from
  `kubernetes-yaml-review` (static manifest review before anything is applied) and
  `kubernetes-security` (security-specific audit) — same domain, three different triggers and
  outputs.
- `argocd-debug` (single Application troubleshooting) is kept distinct from
  `argocd-applicationset` (multi-cluster/environment generator design) and `gitops-review` (repo
  structure and promotion flow) — escalating scope, not duplicated content.
- `root-cause-analysis` (general bug/regression investigation) is kept distinct from
  `incident-analysis` (production incident, audience is a team writeup) — different workflows
  and different output shapes even though both are "investigate first."
- Where two skills' outputs naturally chain (e.g. `architecture-review` finding something that
  needs a `migration-plan`), the skill explicitly points to the other rather than absorbing its
  content.

## How this repository should evolve

- Add one skill at a time, following `CONTRIBUTING.md`'s checklist — check for overlap first,
  keep it narrow, no scripts unless truly unavoidable.
- If a skill's `## Workflow` section grows past what's readable in one sitting, split it instead
  of letting it become a mega-skill.
- Re-run the vetting process in `SECURITY.md`/`docs/third-party-review.md` on any future
  third-party skill import — read fully, grep for risky patterns, extract ideas only, document
  the trust level.
- Prefer `references/` for genuinely reusable supporting material (like the optional-tooling
  tables added to the Kubernetes/Helm/Terraform/GitOps review skills) over bloating `SKILL.md`
  itself — but keep the bar high; most skills should stay flat.
- Revisit `docs/design-notes.md`'s "intentionally not created" list periodically — some of those
  exclusions (day-2 operational skills, CI/CD generation) may become worth a carefully-scoped,
  safety-first skill later, but only as a deliberate addition, never a wholesale import. The
  second batch (`cicd-review` replacing the rejected generator idea with a review-shaped skill)
  is the template for how to do this.
- When adding a skill with real failure modes, add a RED/GREEN scenario pair to `tests/` at the
  same time — the scenarios are how skill edits get regression-checked, and they're cheapest to
  write while the failure mode is fresh in mind.
