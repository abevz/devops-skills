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
- `argocd` (operator/platform umbrella: install/HA/sharding, SSO/RBAC, AppProject, repos, sync
  design, backup/DR across 5 references) is kept distinct from `argocd-debug` (single-Application
  troubleshooting), `argocd-applicationset` (generator design), and `gitops-review` (repo
  structure and promotion flow) — escalating/orthogonal scope, each cross-referencing the others.
- `cilium` (umbrella: design/deploy/configure the many features — install, kube-proxy
  replacement, L3/L4/L7 policy, Hubble, encryption, Cluster Mesh, Gateway/Egress/BGP, across 6
  references) is kept distinct from `cilium-debug` (symptom triage of live drops via Hubble drop
  reasons) — same relationship as `kubernetes-yaml-review` ↔ `kubernetes-debug`. Each
  cross-references the other.
- `istio` (umbrella: install/revision upgrades, sidecar vs ambient data plane, traffic
  management, mTLS/authorization/identity, telemetry — across 5 references) is kept distinct from
  `istio-debug` (503/mTLS/injection symptom triage via Envoy response flags) — the same
  umbrella↔debugger pairing as `cilium`↔`cilium-debug`. Each cross-references the other.
- `observability-stack` (operator umbrella: Prometheus HA/sharding/remote-write, Thanos vs Mimir
  long-term storage, OpenTelemetry Collector pipelines, Loki/Tempo backends — across 4 references)
  is kept distinct from `observability-review` (what to instrument, signals/SLOs), `alert-rule-
  review` (rule quality), and `grafana-dashboards` (viz). The review skill's
  `prometheus-stack.md` reference owns the base kube-prometheus-stack install; the umbrella owns
  the scaling/long-term/pipeline layer on top, and they cross-reference.
- `root-cause-analysis` (general bug/regression investigation) is kept distinct from
  `incident-analysis` (production incident, audience is a team writeup) — different workflows
  and different output shapes even though both are "investigate first."
- Where two skills' outputs naturally chain (e.g. `architecture-review` finding something that
  needs a `migration-plan`), the skill explicitly points to the other rather than absorbing its
  content.

## Relationship to installed third-party skill collections

This repository deliberately does **not** duplicate the third-party collections already
installed and managed separately (via `npx skills add` / symlinks into `~/.agents/skills`).
Upstream sources, as recorded in `~/.agents/.skill-lock.json` at the time of writing:

| Installed collection | Upstream | Role vs. this repo |
|---|---|---|
| `golang-*` (35+ skills) | https://github.com/samber/cc-skills-golang | Go *knowledge base* (idioms, libraries, slog/testify/pprof). The `go-*` skills here are *workflow* skills (fix a bug, review a diff) — complement, not compete. Don't add Go knowledge-base skills here. |
| `terraform-skill` | https://github.com/antonbabenko/terraform-skill | Diagnose-first Terraform knowledge base. `terraform-review` here is the review-workflow counterpart. Both active is intentional. |
| `gws-*` (17 skills) | https://github.com/googleworkspace/cli | Tool-bound (Google Workspace CLI); off-domain. |
| `find-skills` | https://github.com/vercel-labs/skills | Skill discovery tooling; off-domain. |
| `excalidraw-diagram` | https://github.com/coleam00/excalidraw-diagram-skill | Diagramming; off-domain. |
| `beads` | Steve Yegge's beads issue tracker (author field in SKILL.md) | Tool-bound; off-domain. Not in the lock file — installed manually. |
| `chezmoi`, `dream`, `playwright-cli` | not recorded in the lock file (installed manually / bundled) | Tool-bound or off-domain. |

The rule: third-party collections stay upstream-managed and are never vendored in. If one dies
or degrades, write a replacement here in this repo's own shape rather than forking the corpse.

## Depth policy: which skills carry references and which stay flat

Skills split deliberately into two depths:

- **Deep (SKILL.md + references/)** — domains where diagnosis/review quality depends on exact
  facts an agent can't reliably hold: symptom→cause tables, version floors, error-text
  taxonomies, query math. Currently: all `*-debug` skills, `kubernetes-yaml-review`,
  `kubernetes-security`, `helm-review`, `gitops-review`, `terraform-review`, `cicd-review`,
  `observability-review`, `alert-rule-review`, `grafana-dashboards`, `dockerfile-review`,
  `argocd-applicationset`, `production-readiness`, `supply-chain-security`,
  `admission-policy-review`, `dast-review`, `secrets-management`, `cilium` (6 references — its
  own multi-facet umbrella, like the upstream terraform-skill), `argocd` (5 references — operator/
  platform umbrella), `vault` (4 references — Vault/OpenBao operator umbrella), `istio` (5
references — service-mesh operator umbrella), `observability-stack` (4 references — metrics/logs/
traces platform operator umbrella), `cert-manager-debug` (issuance-chain and ACME error-text
playbooks), `ingress` (3 references — north-south umbrella: ingress-nginx EOL, Gateway API,
migration), `kubernetes-autoscaling` (2 references — workload and node layers).
- **Flat (SKILL.md only)** — methodology skills where the workflow itself is the whole content
  and extra reference material would be padding: `git-message`, `pr-review`,
  `root-cause-analysis`, `incident-analysis`, `architecture-review`, `migration-plan`,
  `runbook-writer`, `english-technical-message`, `interview-system-design`,
  `homelab-change-plan`, `vulnerability-triage`, `runtime-security-review`. Also the `go-*`
  trio — deliberately thin because deep Go knowledge lives in the upstream `golang-*`
  collection (see the boundary section above). Note: Go-specific security scanning depth
  (`gosec`, `govulncheck` mechanics) also stays upstream in `golang-security`.

## Review vs deployment content

Most skills are review/design skills — they assume the tool exists and evaluate its use. Where a
control has to be *stood up in the cluster* before it can be reviewed, the skill also carries a
deployment reference with working examples and setup best practices (still markdown-only:
propose Helm values/manifests, never run the install). Current deployment references:
`runtime-security-review/references/falco-deployment.md` (Falco driver/DaemonSet/rules/output),
`admission-policy-review/references/kyverno-deployment.md` (Kyverno controllers/HA/failurePolicy/
CRDs/GitOps), `secrets-management/references/eso-deployment.md` (External Secrets Operator:
provider auth, stores, rotation), and `observability-review/references/prometheus-stack.md`
(kube-prometheus-stack: CRDs, storage, cardinality, provisioning). Tools that are CLI-in-pipeline
rather than in-cluster (cosign, syft, trivy, ZAP) carry their setup as CI-wiring examples in
their existing references instead.

The DevSecOps delivery chain (code → PR → CI → build → registry → GitOps → admission →
runtime → incident response) is covered by composition, not one mega-skill:
`cicd-review` (pipeline attack surface) → `dockerfile-review` (image) →
`supply-chain-security` (SBOM/signing/provenance/registry) → `gitops-review` +
`kubernetes-yaml-review`/`kubernetes-security` (repo and manifests) →
`admission-policy-review` (enforcement) → `runtime-security-review` (detection) →
`vulnerability-triage` (findings process) → `incident-analysis`/`runbook-writer` (response).
`dast-review` (ZAP) attaches at the staging/nightly point for apps and APIs with a web surface,
feeding its findings into `vulnerability-triage` like the other scanners.

When adding a skill, decide its depth explicitly against these criteria rather than defaulting
to flat.

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
- **Conditional references** (`references/conditional/<signal>.md`, loaded only when a signal is
  detected — following upstream kubernetes-skill's pattern) keep cloud/platform-specific depth
  out of the core path. Used for cloud-specific Cilium integration
  (`cilium/references/conditional/{eks,gke,aks}.md` — ENI vs DPv2 vs Azure-CNI-Powered) and
  managed-cluster debugging (`kubernetes-debug/references/conditional/managed-clusters.md`).
  Reach for this when a fact only applies on one platform and would otherwise clutter the
  generic guidance.
- Revisit `docs/design-notes.md`'s "intentionally not created" list periodically — some of those
  exclusions (day-2 operational skills, CI/CD generation) may become worth a carefully-scoped,
  safety-first skill later, but only as a deliberate addition, never a wholesale import. The
  second batch (`cicd-review` replacing the rejected generator idea with a review-shaped skill)
  is the template for how to do this.
- When adding a skill with real failure modes, add a RED/GREEN scenario pair to `tests/` at the
  same time — the scenarios are how skill edits get regression-checked, and they're cheapest to
  write while the failure mode is fresh in mind.
