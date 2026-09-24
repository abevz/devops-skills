# S3 — terraform-review: paired observation

Date: 2026-09-24. Model: baseline `claude-opus-5-5`, with skill `claude-opus-5-5`. Fresh Claude Code sessions; no cluster/infrastructure access or mutation was observed. Score uses [`GREEN criteria`](../../compliance-verification.md) verbatim as the rubric. The answers below are raw model output, not endorsed technical guidance; their factual claims were not independently verified.

## Prompt (identical in both arms)

> I renamed some resources and switched a count to for_each in our RDS module — review the change. You have no repository or infrastructure access; give read-only advice and state what evidence you need.

## Harness

- Baseline: `claude -p --restricted --disable-slash-commands --tools '' --output-format json`.
- With skill: `claude -p --restricted --tools Read --allowedTools Read --add-dir /tmp/devops-skills-paired-candidate` and a system instruction to choose an applicable skill by name/description, then read its SKILL.md and references. The model made four failed path probes before successfully loading the skill. The runtime still advertised other MCP tools despite these flags; the trace shows no mutating tool calls.
- Skill visibly loaded: **yes**. Relevant successful Read trace: `/tmp/devops-skills-paired-candidate/skills/terraform-review/SKILL.md`, `/tmp/devops-skills-paired-candidate/skills/terraform-review/references/state-operations.md`, `/tmp/devops-skills-paired-candidate/skills/terraform-review/references/version-guards.md`, `/tmp/devops-skills-paired-candidate/skills/terraform-review/references/module-design.md`.
- Candidate was assembled from PRs #2–#5 before merge: structure opening plus corrected references where applicable; S4 also has PR #3's final description. This tests a candidate snapshot, not automatic skill discovery from the installed directories.

## Score

| Arm | GREEN criteria | Result |
|---|---:|---|
| Without skill | 3/4 | observed |
| With skill | 4/4 | observed |

**PASS.** The baseline covered identity churn, plan and safety gates, but omitted rollback notes. The skill answer added explicit rollback guidance and met all GREEN criteria.

## Without skill: full answer

> I don't have the diff, so I can't review your actual lines yet. This kind of change has one well-known way to go badly wrong, though, so here's what to check and what I need from you.
>
> ## The main risk: Terraform may destroy and recreate your databases
>
> Terraform tracks each resource by its address. Renaming a resource changes its address, and so does switching from `count` to `for_each`:
>
> - `aws_db_instance.main` → `aws_db_instance.primary` (rename)
> - `aws_db_instance.this[0]` → `aws_db_instance.this["writer"]` (count → for_each)
>
> Unless you tell Terraform that the new address is the same resource, the plan will **destroy the old instance and create a new one**. For RDS that means data loss, or at best a restore from snapshot plus downtime and a new endpoint. The same applies to anything else you renamed in the module: `aws_db_subnet_group`, `aws_db_parameter_group`, `aws_db_option_group`, security groups, `aws_rds_cluster_instance`, and so on.
>
> ## What the change should include
>
> 1. **A `moved` block for every address that changed** (needs Terraform 1.1 or later). Put them in the module:
>    ```hcl
>    moved {
>      from = aws_db_instance.main
>      to   = aws_db_instance.primary
>    }
>
>    moved {
>      from = aws_db_instance.this[0]
>      to   = aws_db_instance.this["writer"]
>    }
>    ```
>    - Every old index needs its own block that maps it to a specific key. If `count = 3` becomes three keys, that's three `moved` blocks.
>    - Renamed module calls in the root module need `moved { from = module.old  to = module.new }`.
>    - Don't use `moved` across state files or providers. It can't do that.
>    - If this is a shared or registry module, keep the `moved` blocks permanently. Callers may upgrade from any older version.
>    - If you're on an older Terraform, you'll need `terraform state mv` instead. That's an out-of-band, hard-to-undo state change, so back up the state first (`terraform state pull > backup.tfstate`) and coordinate it with the apply.
>
> 2. **Stable `for_each` keys.** Use meaningful strings (`"writer"`, `"reader-1"`), not list indexes or values computed at apply time. Keys that come from resource attributes will fail with "Invalid for_each argument". A `toset()` of a list is only safe if the values never change.
>
> 3. **Every reference updated.** Outputs, security group rules, alarms, and DNS records that pointed at `aws_db_instance.x.endpoint` or `aws_db_instance.x[0]` now need `aws_db_instance.x["key"]` or a `for` expression. Splat syntax like `x[*].id` doesn't work on a for_each map. Use `values(x)[*].id`.
>
> 4. **Safety guards as a backstop.** `deletion_protection = true`, `skip_final_snapshot = false` with a `final_snapshot_identifier`, and optionally `lifecycle { prevent_destroy = true }` for the rollout. Don't depend on these alone. `prevent_destroy` stops the plan with an error rather than fixing the addressing.
>
> 5. **No identity changes hidden in the refactor.** If `identifier`, `db_subnet_group_name`, `engine`, `storage_encrypted`, `kms_key_id`, `availability_zone`, or `snapshot_identifier` now come from the new key or name, that forces replacement even when the `moved` blocks are correct. Check that the identifier string ends up exactly the same.
>
> ## Evidence I need to finish the review
>
> | # | What | Why |
> |---|------|-----|
> | 1 | The full diff of the module, plus any root modules or callers that reference it | To check `moved` coverage, keys, and references |
> | 2 | `terraform version` and the pinned provider versions | `moved` needs 1.1+; some provider bumps change forced-replacement attributes |
> | 3 | `terraform plan` output for **every** environment/workspace that uses the module, run against real state | This is the main proof. I'm looking for `has moved to` lines, **0 to destroy**, and no `-/+` or `must be replaced` on any RDS resource |
> | 4 | `terraform state list \| grep -E 'db_\|rds'` from before the change | To confirm every old address has a `moved` target (e.g. `[0]`, `[1]`, … all mapped) |
> | 5 | How the `for_each` map/set is built and what the old `count` expression was | To confirm the index → key mapping is correct in every environment, not just one |
> | 6 | Current `deletion_protection` / `skip_final_snapshot` settings and the backup/snapshot retention | To judge how bad things get if something slips through |
>
> ## Pass/fail rule for merging
>
> Approve only if, in every environment, the plan shows **only moves and in-place updates (`~`)**: no `destroy`, no `-/+`, and no replacement of the DB instance, cluster, subnet group, or parameter group. If any environment shows a replacement, don't apply. Paste that section of the plan and I'll help trace which address or attribute is causing it.
>
> Send items 1 and 3 and I'll do the line-by-line review.

## With skill: full answer

> I couldn't review the change itself because no diff, files or plan were attached. Below is what I'll check once you send them, and what usually breaks when you rename resources and switch `count` to `for_each` in an RDS module.
>
> ## Assumptions
> - Terraform or OpenTofu version, provider versions and `required_version` are unknown. The fixes below need `moved` blocks, which require Terraform 1.1+ (any OpenTofu version). Below that you'd have to use `terraform state mv`, which changes state directly and needs your sign-off first.
> - I'm assuming the module manages `aws_db_instance` or `aws_rds_cluster` plus the resources around it (cluster instances, subnet and parameter groups, `random_password`, security groups).
>
> ## Risk category: identity churn → state corruption / data loss
> Renaming a resource or changing its `count` or `for_each` key changes its address in Terraform. Without `moved` blocks, Terraform destroys the database and creates a new one. For RDS this is the highest-risk kind of change.
>
> ## Findings to verify, in priority order
>
> 1. **Missing `moved` blocks for the renames.** Every renamed resource needs `moved { from = aws_db_instance.old to = aws_db_instance.new }`. Without it the plan shows destroy then create, and the data is lost unless a final snapshot is taken. **Risk:** the production database is replaced. **Fix:** one `moved` block per renamed address, inside the module.
>
> 2. **Missing `moved` blocks for count→for_each.** Each existing instance needs its own move, e.g. `moved { from = aws_rds_cluster_instance.this[0] to = aws_rds_cluster_instance.this["writer"] }`. A block that moves the whole resource doesn't do this. **Risk:** every instance is replaced, and a writer and readers going at the same time means a full outage. **Fix:** one block per index. Check that the key each index maps to is what you intend. In particular, index 0 should map to the current writer.
>
> 3. **Changed arguments vs. changed addresses.** A `moved` block only fixes the address. If the rename also changed `identifier`, `cluster_identifier`, `db_subnet_group_name`, a parameter group `name` or `engine`, the resource is still replaced even with `moved` blocks. **Fix:** keep those values the same, or treat the change as a planned migration.
>
> 4. **Dependent resources are easy to miss.** `random_password`, `random_id` (for example a final-snapshot suffix), subnet and parameter groups, security groups, alarms and `aws_secretsmanager_secret_version` often switched to `for_each` at the same time. If a `random_password` gets a new address without a `moved` block, the master password is regenerated. That's an in-place change, but apps holding the old credential will break.
>
> 5. **`for_each` keys must be known at plan time.** Keys have to come from variables or static locals, not from IDs or ARNs computed by other resources. Otherwise you get `Invalid for_each argument`, and `depends_on` doesn't fix it. A key like `"${var.env}-${count}"` that changes when an input changes brings the replacements back.
>
> 6. **Output shape changes.** `aws_db_instance.this[0].endpoint` or `this[*].endpoint` becomes a map lookup or `values(...)`. If the outputs switch from a list to a map, every caller and every `terraform_remote_state` reader breaks. **Fix:** keep the old output shapes, or add new `_v2` outputs.
>
> 7. **Where the `moved` blocks go.** Inside the module is correct and reaches every caller. If you also renamed the `module "..."` call in any root, that root needs its own `moved { from = module.old to = module.new }`. A `moved` block placed in a module that is being deleted silently does nothing.
>
> 8. **Safety settings.** Check that the change doesn't touch `deletion_protection = true`, `skip_final_snapshot = false` with a `final_snapshot_identifier`, or `lifecycle { prevent_destroy = true }`. These are what turn a bad plan into a failed apply rather than lost data. If any are missing, add them before merging this change.
>
> ## Evidence I need
> - The diff: renamed resources, the old `count` expression and the new `for_each` expression, and any `moved` blocks you added.
> - `terraform plan` (or `tofu plan`) output from **every** root module that calls this module, especially prod. With `-no-color`, redact secrets.
> - `terraform state list | grep -i rds` (or the module path) for each environment, so I can match current indexes to the new keys.
> - `versions.tf`, `required_version` and `.terraform.lock.hcl`.
> - The module's `outputs.tf`, plus a list of the callers and remote-state readers that use it.
>
> ## Validation plan
> 1. Run `terraform validate` and `terraform plan` in each calling root. Don't apply yet.
> 2. For every RDS-related resource, the plan should say "has moved to" and show **zero** `-/+` or `destroy` lines. Any replacement means stop.
> 3. Look for in-place `~` changes on `password`, `identifier` or group names.
> 4. After apply, a plan in each environment should show no changes.
>
> ## Rollback notes
> - Before applying, back up the state (`terraform state pull > backup-<ts>.tfstate`) and take a manual RDS snapshot.
> - If a clean `moved`-only apply causes problems, revert the code and add reverse `moved` blocks. No infrastructure changes.
> - If a replacement slipped through, recovery means restoring from a snapshot to a new endpoint, repointing apps and importing it. That's why the zero-`-/+` check above is the gate. Don't use `-auto-approve` for this change.
>
> Send the diff and the per-environment plans and I'll go through each item concretely.
