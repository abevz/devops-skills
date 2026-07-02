# Third-Party Repository Review

Five repositories were downloaded into the workspace root for inspection and inspiration while
building this skill collection. All five were reviewed read-only — no scripts were executed, no
dependencies were installed, and none of them are tracked by this repository's git history (see
`.gitignore`). They remain on disk locally as untouched reference material.

---

## `agentskills/`

- **Repository**: the canonical [Agent Skills](https://agentskills.io) specification and
  reference tooling.
- **Contents**: `docs/` (Mintlify spec site: `specification.mdx`, `clients.mdx`,
  `skill-creation/`), and `skills-ref/` — a Python package implementing a `SKILL.md` parser and
  validator (frontmatter rules: allowed fields, `name`/`description` constraints). No actual
  skill content — this is spec + tooling, not a skill collection.
- **Useful ideas**: the exact frontmatter/validation rules used throughout this repository
  (lowercase hyphenated `name` matching the directory, `description` starting with "Use when",
  `references/`/`assets/`/`scripts/` conventions, progressive disclosure model) come directly
  from this repo's validator and spec docs.
- **Risky files/patterns**: one `package.json` script (`dev`, a local Mintlify preview server,
  no `postinstall`), one trivial advisory `.claude/hooks/session-start.sh` (checks for a CLI
  tool, prints a suggestion, no side effects). Nothing else.
- **Reused**: Yes — the spec/validator rules shaped every `SKILL.md`'s frontmatter and structure
  in this repository. No files were copied.
- **Rejected**: N/A — no skill content to reject.
- **Trust level**: **High.**

---

## `devops-ai-skill/`

- **Repository**: `qwedsazxc78/devops-ai-skill` — a cross-platform "Horus/Zeus" DevOps skill
  pack (Terraform/Helm + GitOps/Kustomize/ArgoCD), built partly as a conference workshop demo.
- **Contents**: 17 skill directories (12 spec-compliant `SKILL.md`, 5 using a non-standard
  `GUIDE.md` convention), `hooks/` (4 hook scripts), `scripts/` (setup/install tooling per
  agent CLI, Windows mirrors), `prompts/`, multi-tool integration dirs (`.claude/`, `.codex/`,
  `.gemini/`, `.agents/`).
- **Useful ideas**: the plan-only Ingress→Gateway API migration skill family (generate a plan/
  commands file for human review, never execute `kubectl`/`helm` directly) informed the
  `migration-plan` and `gitops-review` skills' safety posture. The hook design (advisory
  `PostToolUse` checks + a single "ask"-gated `PreToolUse` destructive-command guard) informed
  the safety-rules wording used throughout this repository. The optional-local-tooling table
  (`kubeconform`, `kube-score`, `kube-linter`, `polaris`, `pluto`, `conftest`, `checkov`,
  `trivy`, `gitleaks`) informed the `references/tools.md` files added to the Kubernetes/Helm/
  Terraform/GitOps review skills — tool **names and purposes only**, not install commands.
- **Risky files/patterns**: `package.json` has a `postinstall` script (verified safe —
  copy-only, never overwrites, no network calls; **not run**). `scripts/install-tools.sh` is a
  genuinely system-modifying, opt-in script (`sudo apt-get install`, `eval` over a static tool
  list, one official third-party `curl | sh` installer for `uv`) — **not run, not copied**.
  Generator scripts under several skills only *write* `kubectl`/`helm`/`git push` commands to a
  plan file for a human to run; they don't execute them. No secrets, no obfuscation, no
  credential exfiltration found anywhere.
- **Reused**: Partially — methodology and tool names only, as described above. No scripts,
  hooks, or install logic were copied.
- **Rejected**: `install-tools.sh`, `install-global.sh`, `setup*.sh`, `postinstall.js`, and all
  `.claude/`/`.codex/`/`.gemini/` integration glue — none of it was needed for a personal
  markdown-only skill repo.
- **Trust level**: **Medium-High** (no malicious code found; not adopted wholesale because a
  third of its "skills" don't follow the open spec, and it does ship one real system-modifying
  script, even though it's opt-in).

---

## `cluster-skills/`

- **Repository**: `kcns008/cluster-skills` — a Kubernetes/OpenShift skill pack layered with an
  unrelated "cluster-agent-swarm" persona system.
- **Contents**: two overlapping, duplicated skill layers (`skills/` and `.claude/skills/`)
  covering the same Kubernetes ground under different names, plus `.claude/agents/` (6 subagent
  personas with Bash access), `.claude/hooks/`, `cluster-config.*.json` templates.
- **Useful ideas**: some `SKILL.md` descriptions and the shape of security-audit `kubectl`/`jq`
  queries (privileged containers, root containers, etc.) were read as prose reference — reworded
  into the manual checklist in `kubernetes-security`, not copied as scripts.
- **Risky files/patterns**: real, working scripts that run `kubectl apply`, `kubectl drain`,
  `argocd app sync`/`rollback` **without dry-run defaults or confirmation prompts**
  (`onboard-team.sh`, `provision-namespace.sh`, `rollback.sh`). Several scripts reference a
  `shared/lib/preflight.sh` that doesn't exist in the repo — confirming it's an incomplete
  fragment of a larger, un-included codebase. `.claude/hooks/*.sh` are benign/advisory
  (keyword-matching + local file logging). `cluster-config.example.json`/`template.json` contain
  only empty-string placeholders, no real secrets.
- **Reused**: Minimal — a handful of security-check ideas, reworded as prose, no scripts or
  hooks copied.
- **Rejected**: All scripts under `skills/*/scripts/` and `.claude/skills/*/scripts/`
  (cluster-mutating, unsafe defaults, broken dependency), the entire agent-persona layer, the
  three loose top-level `.md` files that break the repo's own stated conventions.
- **Trust level**: **Low-Medium** (no malicious code or embedded secrets, but incoherent,
  duplicated, partially broken, and its mutating scripts lack basic safety defaults — treat as
  idea-source only, never import scripts from it).

---

## `kubernetes-skill/`

- **Repository**: `LukasNiessen/kubernetes-skill` ("KubeShark") — a single, pure-markdown,
  failure-mode-first Kubernetes skill.
- **Contents**: one `SKILL.md` plus `references/` (20 files, ~5,700 lines) covering failure
  modes (insecure defaults, resource starvation, network exposure, privilege sprawl, fragile
  rollouts, API drift), workload patterns, security hardening, observability, multi-tenancy,
  storage, tooling (Helm/Kustomize/policy), and `references/conditional/` for signal-gated
  cloud-provider-specific content. `docs/` is a GitBook mirror. `PHILOSOPHY.md` explains the
  design rationale.
- **Useful ideas**: this was the single richest content source in the batch.
  `references/security-hardening.md` and `references/observability.md` directly informed
  `kubernetes-security`, `kubernetes-yaml-review`, `observability-review`, and
  `production-readiness`. `PHILOSOPHY.md`'s failure-mode framing and progressive-disclosure
  reasoning shaped how `references/` is used sparingly elsewhere in this repository (only where
  it earns its place, per `CONTRIBUTING.md`).
- **Risky files/patterns**: none found. No scripts anywhere in the repo, no hooks, no
  `package.json` scripts of concern (only a `honkit build/serve` for the doc site), no
  curl/wget pipes, no secrets or credential references — verified by full-tree grep.
- **Reused**: Yes — content and structural approach (not files) informed several skills, as
  described above.
- **Rejected**: N/A — nothing risky to reject; this repo was safe to mine liberally.
- **Trust level**: **High.**

---

## `terraform-skill/`

- **Repository**: `antonbabenko/terraform-skill` — a diagnose-first Terraform/OpenTofu skill by
  Anton Babenko (maintainer of the widely-used `terraform-aws-modules` collection).
- **Contents**: one `SKILL.md` plus 8 `references/*.md` files (testing frameworks, module
  patterns, CI/CD workflows, security/compliance, state management, code patterns, LSP
  intelligence, quick reference). `tests/` holds markdown-only RED/GREEN evaluation scenarios
  (`baseline-scenarios.md`, `compliance-verification.md`) and a `rationalization-table.md`
  coverage matrix — a prompt-testing methodology, not executable tests. `mcp.json` optionally
  wires up HashiCorp's official, read-only, Docker-sandboxed `terraform-mcp-server`
  (`autoApprove: []`).
- **Useful ideas**: the strongest single source for `terraform-review`. The "Response Contract"
  pattern (assumptions, risk category, remediation tradeoffs, validation plan, rollback notes)
  was adapted directly into `terraform-review`'s output format, and its discipline (state every
  finding's risk category, never suggest `-auto-approve`) shaped the safety rules of several
  other review skills in this repository. `references/security-compliance.md` and
  `references/state-management.md` informed the workflow checklist.
- **Risky files/patterns**: none found. No scripts, no `postinstall`, no shell files anywhere.
  All `apply`/`destroy` mentions in the docs are either illustrative CI examples or explicit
  anti-patterns the skill tells the agent to avoid; the skill itself hard-codes a rule never to
  run `terraform destroy` without a reviewed `plan -destroy` first.
- **Reused**: Yes — the Response Contract pattern and technical checklist content, not files.
- **Rejected**: N/A — nothing risky to reject.
- **Trust level**: **High.**

---

## `2026-07-02-best-practices-codewhale.md`

Not a repository — a personal research notes file (exported AI conversation) already present in
the workspace. It contains a pre-written brief matching this task almost exactly, including the
target skill list, a directory taxonomy, and an explicit supply-chain vetting checklist (grep
for `curl|wget|bash|sh |rm -rf|sudo|chmod|postinstall|token|secret|kubectl apply|helm install`
before trusting a third-party skill). That checklist is the basis for the "Third-party skills
are untrusted input" section of `SECURITY.md`. Kept locally, gitignored, not part of the
published repository.
