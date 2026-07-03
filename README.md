# DevOps Skills

A small, personal collection of [Agent Skills](https://agentskills.io) for DevOps / Platform /
Kubernetes / GitOps / Go engineering work — Git hygiene, code and PR review, incident and root
cause investigation, Kubernetes and Helm review, ArgoCD/GitOps workflows, Terraform/OpenTofu
review, Go bugfixing and testing, and clear technical English.

Every skill is a plain-markdown `SKILL.md` (plus, for a few, a small `references/` file). There
are no install scripts, no `postinstall` hooks, and nothing here executes commands on your system
on its own — every skill's Safety rules require read-only investigation first and explicit
confirmation before anything mutating.

## Who this is for

Built for a single engineer's daily stack (Kubernetes, ArgoCD, GitOps, Terraform/OpenTofu, Go),
but every skill is self-contained and generic enough to be useful to any DevOps/platform
engineer. Compatible with any agent that supports the Agent Skills format: Claude Code,
Codex-style agents, CodeWhale, OpenCode, and others.

## Safety-first installation advice

- Read a skill's `SKILL.md` before installing it — these are short, plain markdown, this takes a
  minute.
- Prefer installing one skill at a time (`npx skills add ./skills/<name>`) over the whole repo,
  especially at first, so you know exactly what's active.
- None of these skills bundle install scripts or `postinstall` hooks. If you ever add a
  third-party skill from elsewhere, treat it as untrusted input first — see `SECURITY.md`.
- A few review skills mention *optional* local CLI tools (e.g. `kubeconform`, `trivy`,
  `checkov`) in a `references/tools.md` file. These are suggestions only — nothing installs them
  for you, and the skills explicitly instruct the agent not to install or auto-run them.

## Skills

For the top-down view — entry-point flows, umbrella↔debug pairs, the DevSecOps chain, and the
full reference inventory — see [`docs/skill-map.md`](docs/skill-map.md).

| Skill | Category | Purpose |
|---|---|---|
| [`git-message`](skills/git-message) | Git | Write Conventional Commits-style commit messages from a diff |
| [`pr-review`](skills/pr-review) | Review | Lead-engineer-style pull request / diff review |
| [`root-cause-analysis`](skills/root-cause-analysis) | Investigate | Evidence-driven investigation of bugs, regressions, and unclear behavior |
| [`incident-analysis`](skills/incident-analysis) | Investigate | Structured writeup of a production-like incident |
| [`production-readiness`](skills/production-readiness) | Infra | Check whether a service/workload/module is ready for production |
| [`observability-review`](skills/observability-review) | Infra | Review logs, metrics, traces, dashboards, alerts, SLOs |
| [`observability-stack`](skills/observability-stack) | Infra | Operate the stack: Prometheus HA/sharding, Thanos/Mimir, OTel Collector, Loki/Tempo |
| [`architecture-review`](skills/architecture-review) | Architecture | Review repo/system boundaries, coupling, and layering |
| [`migration-plan`](skills/migration-plan) | Architecture | Build a phased, reversible migration plan |
| [`kubernetes-debug`](skills/kubernetes-debug) | Kubernetes | Diagnose a failing Kubernetes workload, read-only first |
| [`kubernetes-yaml-review`](skills/kubernetes-yaml-review) | Kubernetes | Review manifests before they're applied |
| [`kubernetes-security`](skills/kubernetes-security) | Kubernetes | Security audit of workloads, RBAC, and cluster config |
| [`helm-review`](skills/helm-review) | Kubernetes | Review Helm chart templates, values, and upgrade safety |
| [`argocd`](skills/argocd) | GitOps | Install/operate Argo CD: HA, sharding, SSO/RBAC, AppProject, repos, sync design, backup/DR |
| [`argocd-debug`](skills/argocd-debug) | GitOps | Diagnose an out-of-sync or degraded ArgoCD Application |
| [`argocd-applicationset`](skills/argocd-applicationset) | GitOps | Design/review ArgoCD ApplicationSet generators and blast radius |
| [`gitops-review`](skills/gitops-review) | GitOps | Review a GitOps repo's layout, secrets strategy, and promotion flow |
| [`terraform-review`](skills/terraform-review) | Terraform | Diagnose-first review of Terraform/OpenTofu modules and plans |
| [`dockerfile-review`](skills/dockerfile-review) | Containers | Review Dockerfiles for size, security, and reproducibility |
| [`cicd-review`](skills/cicd-review) | CI/CD | Security and reliability review of pipeline definitions (GitHub Actions, GitLab CI) |
| [`cilium`](skills/cilium) | Networking | Design/deploy/review Cilium: install, kube-proxy replacement, L3/L4/L7 policy, Hubble, encryption, Cluster Mesh, Gateway/Egress/BGP |
| [`cilium-debug`](skills/cilium-debug) | Networking | Diagnose Cilium CNI drops and CiliumNetworkPolicy issues via Hubble evidence |
| [`istio`](skills/istio) | Networking | Operate Istio: install/revisions, sidecar vs ambient, traffic management, mTLS/authz, telemetry |
| [`istio-debug`](skills/istio-debug) | Networking | Diagnose Istio mesh issues: sidecars, mTLS, routing, 503 classification |
| [`cert-manager-debug`](skills/cert-manager-debug) | Networking | Diagnose stuck certificates: issuance chain walk, ACME HTTP-01/DNS-01, rate limits, renewal |
| [`grafana-dashboards`](skills/grafana-dashboards) | Observability | Design and review Grafana dashboards as code |
| [`alert-rule-review`](skills/alert-rule-review) | Observability | Write and review Prometheus alert rules (PromQL, noise, coverage) |
| [`runbook-writer`](skills/runbook-writer) | Operations | Write on-call runbooks with read-only diagnosis and marked mutations |
| [`go-bugfix`](skills/go-bugfix) | Go | Fix a Go bug with a minimal, tested change |
| [`go-testing-review`](skills/go-testing-review) | Go | Review Go test quality and coverage gaps |
| [`go-code-review`](skills/go-code-review) | Go | Review Go code for idiomatic style and correctness |
| [`english-technical-message`](skills/english-technical-message) | Communication | Polish short technical English (PR comments, Slack, recruiter replies) |
| [`supply-chain-security`](skills/supply-chain-security) | Security | SBOM, image signing, provenance, deploy-by-digest, registry hygiene |
| [`admission-policy-review`](skills/admission-policy-review) | Security | Kyverno/Gatekeeper/PSA policy design, audit→enforce rollout, exceptions |
| [`vulnerability-triage`](skills/vulnerability-triage) | Security | Turn scanner findings into ranked actions; suppression discipline |
| [`runtime-security-review`](skills/runtime-security-review) | Security | Falco/Tracee coverage and tuning, runtime alert triage |
| [`dast-review`](skills/dast-review) | Security | ZAP DAST scan placement, authenticated/API scans, findings gate |
| [`secrets-management`](skills/secrets-management) | Security | External Secrets Operator / Sealed Secrets / SOPS: deploy, provider auth, rotation |
| [`vault`](skills/vault) | Security | Operate HashiCorp Vault/OpenBao: HA, auto-unseal, auth methods, policies, secret engines, DR |
| [`interview-system-design`](skills/interview-system-design) | Career | Structure and practice system design interview answers |
| [`homelab-change-plan`](skills/homelab-change-plan) | Homelab | Plan homelab infra changes with no staging: tiers, snapshots, bail-out |

## Installing

Install the whole repo:

```bash
npx skills add .
```

Or install a single skill (recommended when trying things out):

```bash
npx skills add ./skills/kubernetes-debug
npx skills add ./skills/argocd-debug
```

Manual copy, if you'd rather not use the installer:

```bash
mkdir -p ~/.agents/skills
cp -r skills/kubernetes-debug ~/.agents/skills/
```

## Suggested daily workflow

```
Plan → Investigate → Patch → Test → Review → Commit
```

- **Plan** — `migration-plan`, `architecture-review` for anything bigger than a one-line fix.
- **Investigate** — `root-cause-analysis`, `kubernetes-debug`, `argocd-debug`, `cilium-debug`,
  `istio-debug`, `incident-analysis`.
- **Patch** — `go-bugfix`, targeted fixes informed by the investigation above.
- **Test** — `go-testing-review` before you consider a fix done.
- **Review** — `pr-review`, `go-code-review`, `kubernetes-yaml-review`, `kubernetes-security`,
  `helm-review`, `terraform-review`, `gitops-review`, `dockerfile-review`, `cicd-review`,
  `observability-review`, `alert-rule-review`, `production-readiness`, as relevant to what
  changed. `grafana-dashboards` and `runbook-writer` close the loop on operability.
- **Commit** — `git-message`, then `english-technical-message` for the PR description or a
  comment explaining the change.

## Recommended skills for this stack

If you only install a handful: `kubernetes-debug`, `kubernetes-security`, `argocd-debug`,
`gitops-review`, `terraform-review`, `production-readiness`, `observability-review`,
`go-bugfix`. These cover the day-to-day Kubernetes/ArgoCD/GitOps/Terraform/Go loop most
directly. Add `cilium-debug`/`istio-debug` if you run those, and `alert-rule-review` +
`grafana-dashboards` + `runbook-writer` if you own the observability stack.

## Repository layout

```
.
├── README.md
├── SECURITY.md
├── CONTRIBUTING.md
├── LICENSE
├── .github/workflows/validate.yml   # CI: frontmatter, naming, no-scripts, no-secrets checks
├── skills/
│   └── <skill-name>/
│       ├── SKILL.md
│       └── references/        # only where genuinely useful: tool lists, bad→good examples
├── tests/
│   ├── baseline-scenarios.md          # RED: unguided-agent failure modes per scenario
│   └── compliance-verification.md     # GREEN: expected behavior with the skill loaded
└── docs/
    ├── design-notes.md
    └── third-party-review.md
```

See `docs/design-notes.md` for why these skills (and not others) were chosen,
`docs/third-party-review.md` for the vetting notes on the third-party repositories this
collection was inspired by, and `tests/` for the markdown-only RED/GREEN methodology used to
check that skills actually change agent behavior.
