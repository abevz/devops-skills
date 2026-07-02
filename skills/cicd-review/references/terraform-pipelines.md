# Terraform/OpenTofu pipelines — plan gates, OIDC, drift, promotion

Distilled from antonbabenko/terraform-skill (`references/ci-cd-workflows.md`) during third-party review.
Use when reviewing CI/CD pipelines that run `terraform`/`tofu` — GitHub Actions, GitLab CI, Atlantis.
Any pipeline step that mutates infrastructure (`apply`, `import`, `destroy`) must sit behind a reviewed plan and an approval gate — never auto-approve from a schedule or an unprotected push.

## Reviewed plan → apply gate (the core pattern)

The apply job must consume the **exact plan artifact that was reviewed** — never re-plan inside apply (state may have moved; the reviewed diff is no longer what executes).

```yaml
# GitHub Actions (trimmed)
plan:
  steps:
    - run: terraform plan -out=tfplan
    - uses: actions/upload-artifact@v4
      with: { name: tfplan, path: tfplan }

apply:
  needs: plan
  if: github.ref == 'refs/heads/main' && github.event_name == 'push'
  environment: production          # protection rules = required reviewers (repo Settings → Environments)
  steps:
    - uses: actions/download-artifact@v4
      with: { name: tfplan }
    - run: terraform init
    - run: terraform apply tfplan  # the saved binary plan, no -auto-approve, no re-plan
```

```yaml
# GitLab CI (trimmed)
plan:
  script: [terraform plan -out=tfplan]
  artifacts: { paths: ["${TF_ROOT}/tfplan"], expire_in: 1 week }
apply:
  script: [terraform apply tfplan]
  dependencies: [plan]
  rules: [{ if: '$CI_COMMIT_BRANCH == "main"', when: manual }]   # manual gate
  environment: { name: production }
```

Review flags:
- ❌ Apply job runs `terraform plan && terraform apply` fresh — the reviewed plan is discarded.
- ❌ `terraform apply -auto-approve` anywhere without a plan-artifact gate; ❌ no environment protection / required approval on production apply.
- ⚠️ A saved plan is bound to the state serial at plan time — if state changed since, apply fails ("saved plan is stale"); that failure is the safety feature, not a bug to work around.
- ⚠️ Plan artifacts and `terraform show -json` output can contain sensitive values — restrict artifact access, set expiry, don't post raw plan JSON to public PR comments.
- ❌ Uncommitted/unreviewed `.terraform.lock.hcl` — CI resolves different provider versions than local ("tests fail in CI, pass locally" ⇒ check `required_version`, provider constraints, lock file before debugging test logic).

## OIDC cloud auth (no static keys)

Keyless OIDC for all three clouds (AWS OIDC roles, Azure federated credentials, GCP Workload Identity Federation). Static keys only if OIDC is unavailable — flag long-lived cloud keys stored as CI secrets.

| Platform | Expected `aud` | Pin `sub` to |
|---|---|---|
| GitHub Actions → AWS | `sts.amazonaws.com` | `repo:<org>/<repo>:ref:refs/heads/<branch>` |
| GitHub Actions → Azure AD | `api://AzureADTokenExchange` | `repo:<org>/<repo>:environment:<env>` |
| GitHub Actions → GCP | value passed via `audience` parameter | repo + ref or environment |
| GitLab CI → AWS | matches `$CI_SERVER_URL` | project path + ref |

```json
"Condition": {
  "StringEquals": { "token.actions.githubusercontent.com:aud": "sts.amazonaws.com" },
  "StringLike":   { "token.actions.githubusercontent.com:sub": "repo:my-org/my-repo:ref:refs/heads/main" }
}
```

- ❌ `sub` wildcards (`repo:*:*`, `repo:<org>/*:ref:*`) — any repo/branch in scope can assume the role. Critical finding.
- ❌ Mismatched `aud` → opaque token-rejected error; fix `aud` per the table — never "fix" by relaxing `sub`.
- Workflow needs `permissions: { id-token: write, contents: read }` for OIDC role assumption.
- ✅ Separate roles per environment; the prod role's `sub` pinned to the protected branch/environment only.

## Drift detection — alert, never auto-apply

✅ Scheduled plan with alert on drift:

```yaml
on: { schedule: [{ cron: '0 */6 * * *' }] }
jobs:
  detect:
    steps:
      - run: terraform init
      - id: plan
        run: terraform plan -detailed-exitcode -out=plan.bin
        continue-on-error: true
      - if: steps.plan.outcome == 'failure' && steps.plan.outputs.exitcode == '2'
        run: echo "Drift detected — requires human review"   # → Slack/PagerDuty/issue
```

`plan -detailed-exitcode`: `0` = no drift, `1` = plan failed, `2` = drift detected.

❌ **Anti-pattern the source explicitly flags:** scheduled `terraform apply -auto-approve` on cron as "drift remediation" — silently reverts out-of-band changes (including emergency manual fixes) with no review. Rate as CI-drift risk. Same verdict for "self-healing" GitOps-style auto-apply of Terraform without a plan gate.

## PR plan visibility

- ✅ Plan runs on `pull_request`, result surfaced to reviewers: plan artifact + summary, or Atlantis PR comments, or Infracost cost comment (`infracost breakdown --path . --format json` + comment action with `behavior: update`).
- Atlantis: plan/apply driven by PR comments, per-project config in `atlantis.yaml` (pins `terraform_version` per project), built-in locking prevents concurrent PRs planning the same project; plan step may use `-lock=false` since PR plans are advisory.
- ✅ Validate stage on every PR before plan: `fmt -check -recursive`, `init`, `validate`, `tflint` (pinned version, `tflint --init`), plus security scan (trivy config / checkov — see terraform-review `security-compliance.md` for placement).
- ⚠️ For fork PRs, plan jobs must not receive cloud credentials/secrets (combine with the `pull_request_target` checks in the main skill).

## Multi-environment promotion shape

- Per-env workflows (`terraform-dev.yml` / `-staging.yml` / `-prod.yml`) or one reusable workflow (`workflow_call` with `environment` input, `environment: ${{ inputs.environment }}` binding protection rules per env).
- ✅ Promotion order dev → staging → prod, same code, different composition/tfvars; prod gated by required reviewers on the environment.
- ✅ Per-env state keys and per-env OIDC roles — a dev pipeline must be physically unable to touch prod state.
- ❌ One workflow with matrix envs sharing a single cloud role/state path.
- ✅ Provider/runtime upgrades promoted separately from functional changes (separate PRs), test dev → stage → prod.

## Concurrency and locking in CI

| Layer | Mechanism |
|---|---|
| Workflow level (GH Actions) | `concurrency: { group: terraform-${{ github.ref }} (or per-env), cancel-in-progress: false }` — queue, don't cancel a running apply |
| Job level (GitLab) | `resource_group: terraform-prod` — one job at a time |
| State level | backend locking (S3 `use_lockfile = true` on TF/OpenTofu 1.10+; DynamoDB pre-1.10) + `terraform apply -lock-timeout=10m tfplan` to wait instead of failing instantly (default `-lock-timeout=0s`) |

- ❌ `cancel-in-progress: true` on a group containing apply jobs — killing an apply mid-run is where stale locks and corrupted state come from.
- ❌ Auto-`force-unlock` in pipeline scripts — locks must fail the job for a human; repeated stale locks mean the job timeout is shorter than apply duration (fix that instead).
- Per-PR isolated state: backend blocks support no interpolation — inject at init: `terraform init -backend-config="key=pr-${PR_NUMBER}/terraform.tfstate"`.

## Cost and hygiene

- ✅ Mocked/unit tests (`terraform test`, mock providers 1.7+) on PRs (free); real-cloud integration tests only on main branch or schedule.
- ✅ Tag every test-created resource (`Environment=test`, `CreatedAt` RFC3339, `JobID`) + scheduled cleanup job (cron + `workflow_dispatch`) that terminates test resources older than a cutoff — note AWS tag filters are equality-only, so timestamp filtering happens client-side (jq).
- ✅ Unique names per test run (run ID + random suffix) — parallel-run collisions surface as `ResourceAlreadyExists`.
- ✅ Plugin caching: set `TF_PLUGIN_CACHE_DIR` env **and** cache that path keyed on `hashFiles('**/.terraform.lock.hcl')` — caching without the env var caches nothing.
- ✅ Pin CI tool versions: setup actions with explicit versions, `tflint_version`, scanner actions pinned — and `required_version` + provider constraints in code so CI matches local.
