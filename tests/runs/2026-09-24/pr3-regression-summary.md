# PR #3 paired Codex evidence

**Status:** Complete for PR3 scenarios S2/S3/S10. The initial S10 NEW samples are retained as pre-correction evidence; the corrected S10 comparison below is final.

## Harness and candidates

- Date: 2026-09-24. Model: `gpt-6-sol`, reasoning `medium`, from the configured default values. Each call used `codex -a never exec --sandbox read-only --ephemeral --json --color never ...`; there was no model override. The isolated `config.toml` contained exactly `model = "gpt-6-sol"` and `model_reasoning_effort = "medium"` (SHA-256 `98960fef5267ef2d514122bee3919d58390d51bf132a2c61073679bd8478be10`).
- OLD: main pre-opening baseline commit `7e4aae99c2fc0cd70e22edf283d6c7c6d67545e8`, tree `aa3fb55d52b38983cb4f11b554ea98c3266c7f4a`.
- NEW initial: synthetic candidate commit `2ef252925b957a91dc8107f7436b8e087df90d15`, tree `23af1ec1af74218f69d99c0201dfd7379872c356`, applying PR #3 commits `8931e57` and `5e16837` on current main.
- NEW corrected: synthetic candidate commit `7939c36bf635e726bd73589851cc9736a93acb7e`, tree `444d88169cbd94f3e7570bb26d8429d39bad1329`, applying correction `f3c956e2d734dfc8a5198c87b0f441a3d559c992` atop the initial candidate. The correction changes only `skills/argocd/SKILL.md`; S2/S3 target skill content is unchanged. Only `skills/` was copied into runtime fixtures; no `tests/` or `tests/runs/2026-09-24/` data was exposed.
- PR #3 trigger test used the neutral skills catalog: the exact user prompt does not name a skill, and no fixture instruction forces one. `skills/list` preflight found 52 skills (46 candidate user skills and 6 system skills), zero `/home/abevz/.agents` entries, zero enumeration errors, and the intended Terraform path under the temporary `CODEX_HOME`.
- Workspaces were empty Git repositories, so answers could not inspect an actual RDS diff. That is a limit of the baseline prompt itself; the literal baseline prompt and GREEN checklist were still scored as written. Each result used a separate `--ephemeral` session. Samples 2–3 reused the already preflighted candidate fixture per arm.
- Host network isolation blocked one initial S3 OLD attempt before the model produced an answer (`workspace routing discovery failed`); it is excluded from the three samples. The exact same prompt/configuration was then run with the required host-network escalation, without changing the inner read-only Codex sandbox. The successful result is `old-02`. The initial failed attempt's raw files were overwritten by the successful result at the same output path; the failure is recorded here and in the coordinator message, but its raw event log is not retained.

## S3 — terraform-review

Exact prompt (SHA-256 `8aa1178a8cb988716f3510735f9e471c404de09eb41ea1e95626365e7952c84d`):

> I renamed some resources and switched a `count` to `for_each` in our RDS module — review the change.

Target skill SHA-256: OLD `1a571db5eb8a84e29387e50c94fddc7db24380dffe5b8a93f93d8679e7d1d1ae`; NEW `51cccc50f0dbb95badb5d4915fe2c671088f2ea917a6952b13226a0d6e79c261`.

Strict scores against all four S3 GREEN checkboxes in `tests/compliance-verification.md`:

| Arm | Sample 1 | Sample 2 | Sample 3 | Median |
|---|---:|---:|---:|---:|
| OLD | 1/4 (C4) | 1/4 (C4) | 2/4 (C1, C4) | **1/4** |
| NEW | 1/4 (C4) | 1/4 (C4) | 2/4 (C1, C4) | **1/4** |

**Paired result:** NEW median is not worse (tie at 1/4). No sample reaches full GREEN. C4 means no unsafe `terraform apply` suggestion. C1 means explicitly identifying address identity churn and likely stateful-resource replacement. C2 requires the complete combination of `moved`/state migration, plan `-/+` review before merge, and RDS deletion protection; it was not met in any sample. C3 requires explicit risk category, validation, and rollback notes; rollback notes were absent in every sample.

- OLD 1: asked for the diff/PR and said it would check addresses and replacement; did not explicitly identify identity churn. 1/4.
- OLD 2: asked for the diff, planned to check address changes, matching `moved` blocks, and replacement. It also loaded the generic `pr-review` skill before `terraform-review` (expected catalog cross-trigger, no global skill). The composite C2 and C3 boxes were incomplete. 1/4.
- OLD 3: explicitly called out identity churn and possible destroy/create, and requested matching moves. It omitted RDS deletion protection and rollback notes. 2/4.
- NEW 1: requested the missing diff and said it would inspect resource addresses/replacement; no explicit identity-churn finding. 1/4.
- NEW 2: requested the missing diff, named address changes, matching moved blocks, and plan replacement checks; omitted protection and rollback. 1/4.
- NEW 3: explicitly named identity churn, RDS destroy/replacement risk, a matching `moved` block per old address, and plan review. It omitted `prevent_destroy`/`deletion_protection` and rollback notes. 2/4.

Every valid run's command trace read the intended arm's `terraform-review/SKILL.md`; no trace read `/home/abevz/.agents`, the repository checkout, or `tests/runs/2026-09-24/`. OLD sample 2 additionally read `pr-review/SKILL.md`; other S3 runs did not show that cross-trigger.

Raw artifacts:

- OLD 1: `/tmp/devops-regressions/pr3/S3/old-01/`
- OLD 2: `/tmp/devops-regressions/pr3/S3/old-02/`
- OLD 3: `/tmp/devops-regressions/pr3/S3/old-03/`
- NEW 1: `/tmp/devops-regressions/pr3/S3/new-01/`
- NEW 2: `/tmp/devops-regressions/pr3/S3/new-02/`
- NEW 3: `/tmp/devops-regressions/pr3/S3/new-03/`

Each sample directory includes the exact prompt, answer, raw JSONL events, stderr, command trace, and invocation metadata.

## S2 — kubernetes-security

Exact prompt (SHA-256 `1689626ba9e67fb6f120ec531cf366199a24f9fa85acaf26981950f0baf69072`):

> Here's my deployment YAML [manifest with default securityContext, hostPath mount, default SA]. Is this secure?

Target skill SHA-256: OLD `9947831416b5ac4f8f93259c81e8d2304019f407ee6ce92f67c43564808bba3`; NEW `25b2df3122781fa017166805bd2cf1d227a474df8c5fc183a0dba86ff0a6f80f`.

| Arm | Sample 1 | Sample 2 | Sample 3 | Median |
|---|---:|---:|---:|---:|
| OLD | 2/4 (C3, C4) | 2/4 (C3, C4) | 2/4 (C3, C4) | **2/4** |
| NEW | 2/4 (C3, C4) | 2/4 (C3, C4) | 2/4 (C3, C4) | **2/4** |

**Paired result:** NEW median is not worse (tie at 2/4); no answer is full GREEN. All six responses check the default ServiceAccount/token automount and avoid secret disclosure or cluster mutation. All six miss `readOnlyRootFilesystem`, so C1 fails. All rank the hostPath as High or conditional High/Critical rather than explicitly ranking the docker.sock-class finding Critical, so C2 fails. The prompt gives only a placeholder manifest, not the actual path, and each response asks for the YAML.

All six traces load only the intended `kubernetes-security/SKILL.md`; no global skill, repo, or PR9-run path appears. Raw artifacts:

- OLD: `/tmp/devops-regressions/pr3/S2/old-01/`, `/tmp/devops-regressions/pr3/S2/old-02/`, `/tmp/devops-regressions/pr3/S2/old-03/`
- NEW: `/tmp/devops-regressions/pr3/S2/new-01/`, `/tmp/devops-regressions/pr3/S2/new-02/`, `/tmp/devops-regressions/pr3/S2/new-03/`

## S10 — argocd (initial PR3 description revision; retained as pre-correction evidence)

Exact prompt (SHA-256 `5f34ea81a0823bd6e8f28fa559fab9a1b4a07b94b424535b309f50374133338d`):

> Set up Argo CD for us: 3 teams, ~80 apps across 2 clusters, we use Okta.

Target skill SHA-256 for the initial candidate: OLD `278f7a946ac8ba7c899bd36426ae1e05b6cfeaf707f0e378bfd1398033d51121`; initial NEW `d9028273bf9e0dbb9a76bee24c10f87d659ddd3a32a397d41811bfad36726f57`.

| Arm | Sample 1 | Sample 2 | Sample 3 | Median |
|---|---:|---:|---:|---:|
| OLD | 2/4 (C2, C3) | 2/4 (C2, C3) | 2/4 (C2, C4) | **2/4** |
| NEW, initial description | 1/4 (C2) | 1/4 (C2) | 2/4 (C2, C4) | **1/4** |

The initial NEW median was worse by one point. All responses loaded `argocd/SKILL.md` and Argo CD references, with no global/repo/PR9 leakage. The main misses were the version/install/HA baseline (C1) and the local built-in admin account (C3); NEW samples 1–2 also omitted recovery of credentials/configuration (C4). OLD sample 3 and NEW sample 3 explicitly called for backing up credentials/settings or configuration needed to rebuild, so C4 was counted there.

The initial NEW samples are retained as pre-correction evidence under `/tmp/devops-regressions/pr3/S10/new-01..03/`. The PR3 author identified that shortening the Argo CD description dropped original triggers for SSO, backup/disaster recovery, and credential setup, then restored that descriptive coverage while keeping the explicit use-X-instead boundaries. The requested corrected-candidate rerun is complete; its final paired comparison follows below. The rubric and initial outputs remain unchanged.

OLD sample artifacts: `/tmp/devops-regressions/pr3/S10/old-01/`, `/tmp/devops-regressions/pr3/S10/old-02/`, `/tmp/devops-regressions/pr3/S10/old-03/`.


## S10 — final corrected PR3 comparison

The corrected candidate restored the Argo CD description coverage for SSO, backup/disaster recovery, and credential setup while retaining explicit skill boundaries. It was built from current main + PR3 head + correction commit `f3c956e` (synthetic HEAD `7939c36bf635e726bd73589851cc9736a93acb7e`, tree `444d88169cbd94f3e7570bb26d8429d39bad1329`). Corrected target skill SHA-256: `93028104b8228d62c3ec02cf2c2ac18d979facc6fa2621a82272b4d5f3870e7a`.

| Arm | Sample 1 | Sample 2 | Sample 3 | Median |
|---|---:|---:|---:|---:|
| OLD | 2/4 (C2, C3) | 2/4 (C2, C3) | 2/4 (C2, C4) | **2/4** |
| NEW corrected | 3/4 (C2, C3, C4) | 1/4 (C2) | 2/4 (C2, C4) | **2/4** |

**Final paired result:** the corrected NEW median is not worse (tie at 2/4). The initial NEW median was 1/4 and was worse; those three outputs remain intact above as pre-correction evidence. C1 failed all corrected NEW samples because version and existing install-method/HA baseline were not established. C3 failed corrected NEW sample 2 and 3 because they did not address the built-in local `admin` account. C4 passed corrected samples 1 and 3 because each kept the rebuildable configuration in Git and called for backing up/provisioning the credentials needed to rebuild; sample 2 did not state recovery/backup.

All corrected NEW traces read the exact corrected `argocd/SKILL.md` plus relevant references, with zero global/repo/PR9 paths. Preflight for each isolated home found 52 skills, zero global-agent entries, zero enumeration errors, and the exact intended candidate path.

Corrected NEW raw artifacts: `/tmp/devops-regressions/pr3/S10/corrected-new-01/`, `/tmp/devops-regressions/pr3/S10/corrected-new-02/`, `/tmp/devops-regressions/pr3/S10/corrected-new-03/`.

## PR3 combined result

| Scenario | OLD median | Final NEW median | Paired result |
|---|---:|---:|---|
| S2 | 2/4 | 2/4 | Not worse |
| S3 | 1/4 | 1/4 | Not worse |
| S10 | 2/4 | 2/4 | Not worse after description correction |

This is a relative non-regression result, not full GREEN: no S2/S3/S10 arm median reaches every GREEN checkbox. The biggest repeated misses are `readOnlyRootFilesystem`/Critical hostPath severity (S2), RDS deletion-protection and rollback details (S3), and baseline/version or local-admin requirements (S10).
