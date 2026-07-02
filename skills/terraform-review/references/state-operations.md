# State operations — surgery, migration, drift, recovery

Distilled from antonbabenko/terraform-skill (`references/state-management.md`) during third-party review.
Use when reviewing or advising on state refactors, backend changes, drift, lost/corrupted state, or destroys.
Every state-mutating command below (`state mv`, `state rm`, `state push`, `import`, `force-unlock`, `apply`)
**requires explicit user confirmation and a plan review first** — never run or recommend them as automatic steps.

## State surgery decision table

| Goal | Use | Floor (Terraform / OpenTofu) | Why / tradeoff |
|---|---|---|---|
| Rename resource or module, move into `for_each` keys | `moved` block | 1.1+ / all | Declarative, code-reviewed, plan shows "moved" not destroy/create |
| Adopt existing/unmanaged resource into state | `import` block (+ `terraform plan -generate-config-out=gen.tf`) | 1.5+ / 1.6+ | Reviewable in VCS; prefer over CLI import |
| Adopt resource, below import-block floor | `terraform import <addr> <id>` | any | CLI state mutation — **confirm first**, write matching config before importing |
| Stop managing a resource but keep it alive | `removed` block with `lifecycle { destroy = false }` | 1.7+ / 1.7+ | Declarative; resource becomes unmanaged, not destroyed |
| Same, below floor | `terraform state rm <addr>` | any | Orphans the real resource — **confirm first**; only when intentionally abandoning |
| Refactor when `moved` unavailable (pre-1.1) or ad-hoc | `terraform state mv <from> <to>` | any | Mutation, not code-reviewed — **confirm first**; prefer `moved` on 1.1+ |
| Move resource to a *different* state file | `state rm` in source + `import` in destination | any | `state mv` and `moved` **cannot cross state files/backends** |
| Destroy the real resource | delete config → `plan -destroy` review → `apply` | any | See safe-destroy protocol below |

**`moved` block limits** (all are common review findings):
- ❌ Cannot cross a provider boundary → use `removed` (1.7+) + `import` (1.5+) instead.
- ❌ Cannot cross a state file / backend key → `state rm` + `import` with backups.
- ❌ A `moved` block *inside* a module being deleted silently no-ops and resources get destroyed → put the `moved` block in the **parent** module.
- ❌ Rename without a `moved` block = destroy/create. Verify plan shows a move operation, not `-/+`.

**Before any surgery:** `terraform state pull > backup-$(date +%Y%m%d-%H%M%S).tfstate`. After: `terraform plan` must show no changes.

## Backend migration procedure

1. Backup: `terraform state pull > pre-migration.tfstate` (and keep local `terraform.tfstate` copies).
2. Add/replace the `backend` block in code.
3. `terraform init -migrate-state` — answer yes to "copy existing state". (`-reconfigure` instead creates a **new empty state** — only for deliberate re-import/rebuild.)
4. Verify: `terraform plan` shows **no changes**; confirm the object exists in the new backend.
5. Keep old state as backup 30–90 days before deleting. Commit the backend config.

Terraform Cloud specifics: `terraform login` first; the workspace auto-creates on first `init` against the `cloud {}` block. ❌ Do NOT run `terraform workspace new` — CLI workspaces are a different concept from TFC workspaces.

DynamoDB → S3 native lock migration (1.10+): set **both** `dynamodb_table` and `use_lockfile = true` during transition (locks acquire via both); drop `dynamodb_table` once every workflow runs 1.10+. Flag configs that add a DynamoDB lock table on 1.10+ runtimes — `use_lockfile = true` replaces it.

Backend config rules: ✅ partial config (`-backend-config=` at init) for sensitive values — backend blocks support **no interpolation or env vars**. ✅ Per-PR state keys in CI via `terraform init -backend-config="key=pr-${PR}/terraform.tfstate"`.

## State splitting / merging (moving resources between root modules)

`terraform state mv` only works **within one state file**. To move `aws_rds_cluster.main` from stack A to stack B:

1. Backup both states (`state pull` in each).
2. In destination: add config + `import` block (or `terraform import aws_rds_cluster.main <cluster-id>` — **confirm first**), then `plan` until zero diff.
3. In source: only after the destination plan is clean, `terraform state rm aws_rds_cluster.main` (**confirm first**) and delete the config.
4. `terraform plan` in both stacks must show no changes. Order matters: import before rm, or a plan in either stack can schedule a destroy.

When to split state: different lifecycles (VPC vs apps), different owning teams, different risk profiles, >500–1000 resources (slow operations — heuristic, depends on provider refresh), independent deploy cadence. Combine when: tightly coupled resources, same lifecycle, <100 resources, single team.

## Drift detection and reconciliation

| Task | Command | Effect |
|---|---|---|
| Detect drift (read-only) | `terraform plan -refresh-only` (0.15.4+) | Shows infra-vs-state differences, changes nothing |
| Detect drift for automation | `terraform plan -detailed-exitcode` | Exit 0 = clean, 1 = plan error, 2 = drift |
| Accept reality into state | `terraform apply -refresh-only` | Updates **state only**, no infra changes — **confirm first** |
| Revert infra to code | regular `plan` review → `apply` | Changes infrastructure — **confirm first** |

Common causes: console/CLI manual changes, other tools (CDK, scripts), out-of-band deletion. Prevention: scheduled `plan -detailed-exitcode` in CI that **alerts** (never auto-applies — see cicd-review `terraform-pipelines.md`), CloudTrail on manual changes, termination protection on critical resources.

After external secret rotation: `terraform apply -refresh-only` reconciles state to the new value — it does not rotate anything itself.

## Disaster recovery

**State corrupted / bad write** — versioned backend makes this recoverable:
```bash
aws s3api list-object-versions --bucket my-terraform-state --prefix prod/terraform.tfstate
aws s3api get-object --bucket my-terraform-state --key prod/terraform.tfstate \
  --version-id PREVIOUS_VERSION terraform.tfstate.restored
terraform state push terraform.tfstate.restored   # overwrites remote state — CONFIRM FIRST
terraform plan                                    # then reconcile drift with apply -refresh-only
```

**`state snapshot was created by Terraform vX, newer than current vY`** — not corruption: someone ran a newer CLI. Fix by upgrading the runtime (e.g. `tfenv install/use`), not by editing state.

**State completely lost, no backup** — infrastructure is intact; state is rebuildable:
- `import` blocks (1.5+/1.6+) + `terraform plan -generate-config-out=generated.tf`, review and merge generated config, iterate until `plan` is clean. Below floor: `terraform import` per resource (**confirm each**).
- Recoverable: everything a provider can read by ID. Lost forever: values only Terraform knew (e.g. `random_password` results, unexported attributes) — those resources need secret rotation or recreation.

**Recoverability checklist** (review flag if missing): versioning on the state bucket, documented backend config, restore procedure actually tested, encryption keys accessible.

❌ Never hand-edit `terraform.tfstate` (JSON surgery in an editor) — use `state mv/rm`/`import`/`push` with backups. ❌ Never commit `*.tfstate` to git.

## Workspaces vs directory-per-environment

| Factor | CLI workspaces | Directory per env (recommended default) |
|---|---|---|
| State separation | Same backend, same credentials, key suffix | Separate backend keys, separate IAM roles per env |
| Blast radius | Wrong `workspace select` applies dev code to prod | Physically separate roots; hard to cross |
| Config divergence | One codebase, `terraform.workspace` conditionals creep in | Explicit per-env `tfvars`/composition |
| Access control | ❌ Cannot scope IAM per workspace | ✅ IAM path-scoped per env (`prod/*` vs `dev/*`) |

❌ Workspace-only isolation is not a substitute for backend-level IAM separation. ❌ Never mix prod and non-prod under the same backend key/credentials. Recommended layout: `<env>/<component>/terraform.tfstate` keys (numbered components — `01-networking`, `02-platform` — encode dependency order).

## `terraform_remote_state` coupling risks

- ❌ Default glue inside a single team's stack — wire via module outputs instead.
- ❌ Chains of >2 remote-state reads in one composition — hidden cross-stack coupling; reshape boundaries.
- ❌ Reading provider-queryable values — prefer cloud data sources (`aws_vpc` by tag) which can't go stale against a stale state.
- ❌ Consumers get **read access to the entire producer state file**, including any secrets in it — SSM Parameter Store / explicit exported values give finer-grained IAM.
- ✅ Legitimate only at true ownership boundaries: different teams/release cadences, state already split for lifecycle reasons, values can't be passed as inputs.
- ✅ Producers: treat outputs as a versioned contract — document external consumers, never rename casually.

## Safe destroy protocol

A targeted destroy cascades beyond its targets via implicit dependencies:

1. `terraform plan -destroy [-target=...]` — never go straight to destroy. **Requires user confirmation before any apply.**
2. Read **every** resource under "will be destroyed", not just the targets.
3. ⚠️ `for_each` resources fed by a `locals {}` value that references a targeted resource become implicit dependents — targeting one `aws_eip` referenced in a local used by `for_each` DNS records destroys **all** those records.
4. Get explicit user confirmation of the full destroy list.
5. ❌ Never `-auto-approve` on destroy; never present it as acceptable in production.

Provider removal ordering: ❌ removing the `provider` block before its resources are destroyed/removed → hard error (plan can't resolve the type). ✅ Two phases: destroy/remove resources with the provider still configured, verify `state list` is empty for that provider, *then* delete the provider block and re-run `terraform init`.

Stuck lock handling (`force-unlock`) lives in `version-guards.md` — verify the operation is genuinely dead before unlocking; unlocking under a live apply corrupts state.
