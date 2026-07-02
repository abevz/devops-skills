# Module design — sizing, contracts, iteration, anti-patterns

Distilled from antonbabenko/terraform-skill (`references/module-patterns.md`, plus unique material from `references/code-patterns.md`) during third-party review.
Use when reviewing module structure, variable/output contracts, `for_each`/`count` choices, or module versioning.

## Module sizing and composition

| Level | Scope | Reusable? | Example |
|---|---|---|---|
| **Resource module** | One logical group of tightly coupled resources | High | VPC + subnets; SG + rules; IAM role + policies |
| **Infrastructure module** (wrapper/facade) | Orchestrates several resource modules for one purpose | Moderate | web-app = vpc + alb + ecs wired together |
| **Composition** (root module) | Complete environment; concrete values, backend config | No — env-specific | `environments/prod/` |

Classification: env-specific config → composition; combines multiple concerns → infrastructure module; focused resource group → resource module.

Rules:
- ✅ Smaller scopes = faster plan/apply, smaller blast radius, parallel team work. Split a 2000-line prod root into `networking/`, `compute/`, `data/`, `storage/`, `iam/`.
- ✅ Compositions hold environment-specific values (`instance_class = "db.r5.xlarge"`, `multi_az = true`); modules hold abstractions.
- ✅ `terraform.tfvars` and `backend.tf` exist **only** at composition level, never inside reusable modules.
- ✅ Standard module files: `main.tf`, `variables.tf`, `outputs.tf`, `versions.tf`, `README.md`; plus `examples/{simple,complete}/` and `tests/` for published modules (Registry requires this layout).
- The facade pattern is the infrastructure module: it adds a real contract (wiring, opinionated defaults) on top of resource modules — see thin-wrapper anti-pattern below for when a facade adds nothing.

## Variable design

Block ordering: `description` (always) → `type` → `default` → `sensitive` → `nullable` → `validation`.

```hcl
variable "environment" {
  description = "Environment name for resource tagging"
  type        = string
  default     = "dev"
  nullable    = false                     # 1.1+: null falls back to default instead of overriding it

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be one of: dev, staging, prod."
  }
}
```

| Rule | ❌ | ✅ |
|---|---|---|
| Typing | `map(any)` / `any` for core module contracts | typed `object({...})` with `optional(attr, default)` (Terraform 1.3+) |
| Naming | generic `var.name`, `var.type`, `var.port` | context-specific `var.vpc_cidr_block`, `var.application_port` |
| Nulls | omit `nullable`, letting `null` silently override defaults | `nullable = false` (1.1+) on vars that must never be null |
| Secrets | assume `sensitive = true` keeps value out of state | it only masks display — see `security-compliance.md` |
| Complexity | `object()` for everything | simple types unless strict validation is needed; `any` only to deliberately disable validation |

- ✅ Cross-variable validation (`condition` referencing another `var.*`) — Terraform 1.9+; below the floor, use resource `precondition`.
- ✅ Every reusable module's `validation` blocks must be exercised by tests — reject cases matter as much as happy paths.

### Validation mechanism timing (routinely confused)

| Mechanism | Runs | Can reference | Blocks apply? |
|---|---|---|---|
| `validation` in `variable` | before plan | own value; other vars on 1.9+ | yes |
| `precondition` | before resource create/update | resources, data sources, vars | yes |
| `postcondition` | after apply | resource's own computed attrs | yes |
| `check` block (1.5+) | every plan+apply | anything | **NO — advisory warnings only** |

Flag any review where a `check` block is expected to gate an apply.

## Output contracts

- ✅ Always `description`; `sensitive = true` on secrets; return objects to group related values (`{ id, private_ip, public_dns }`).
- ✅ Naming `{name}_{type}_{attribute}`; plural for lists (`private_subnet_ids`); ❌ no `this_` prefix in output names.
- ✅ `try(aws_security_group.this[0].id, "")` for conditional resources (0.12.20+) — ❌ legacy `element(concat(...))`.
- ✅ Outputs consumed by other stacks are a versioned contract: document external consumers, add a `_v2` rather than breaking rename.
- ❌ Exposing whole resource/provider objects as outputs — leaks the entire contract; export a stable subset.

## Version pinning

| Component | Recommended | Note |
|---|---|---|
| Terraform runtime | `required_version = "~> 1.9"` | minor pin, patch updates allowed |
| Providers | `version = "~> 5.0"` | major pin; `~>` increments rightmost component only (`~> 5.0` = <6.0; `~> 5.0.1` = 5.0.x patches) |
| Module sources (prod) | exact `version = "5.1.2"` | dev may use `~> 5.1` |
| Lock file | commit `.terraform.lock.hcl` | `terraform init -upgrade` to bump within constraints, then plan-review |

- ❌ Floating module sources (no `version`) in consumer code — upstream major bumps break silently.
- ❌ `>= 5.0` alone or no constraint in production.
- ❌ Merging provider/runtime upgrades with functional changes in one PR.

## Multi-provider modules (aliases)

- ✅ Child declares aliases in `required_providers`: `configuration_aliases = [aws.primary, aws.replica]`; binds per resource with `provider = aws.primary`.
- ✅ Caller passes `providers = { aws.primary = aws.us_east_1, aws.replica = aws.eu_west_1 }` on the `module` block.
- ❌ Omitting the `providers` map on multi-region/multi-account calls — plan fails with "No configuration for provider aws.primary", or resources silently land on the default provider. Default inheritance applies **only** to a single unaliased provider.

## for_each / count keying rules

| Case | Use | Why |
|---|---|---|
| Collection with meaningful identity | `for_each` keyed by stable business value (AZ name, env name) | removing/reordering one element leaves other addresses untouched |
| Optional singleton | `count = var.create_x ? 1 : 0` | boolean condition, 0-or-1 |
| Simple numeric replication, items never removed from middle | `count` acceptable | |
| Keys unknowable at plan time | `count` singleton or restructure inputs | `for_each` keys must resolve during plan |

- ❌ `count` over a list where middle items can be removed — every subsequent resource is destroyed/recreated (index churn).
- ❌ `for_each` keys derived from another resource's computed attrs (IDs/ARNs) — `Invalid for_each argument`; `depends_on` does **not** fix this (it orders applies, not plan-time resolution). ✅ Drive keys from user variables or static locals.
- ❌ `count.index` as long-lived identity — use business-meaningful `for_each` keys.
- Migration count→for_each: add `for_each`, emit one `moved { from = res[0] to = res["key"] }` per instance (1.1+), verify plan shows moves not `-/+`. Omitting `moved` during any rename/refactor is the classic silent destroy/create.

## Dynamic blocks

- ❌ Bare `dynamic "ingress"` nested inside a `for_each` resource — the block-name iterator (`ingress.value`) shadows the outer `each`. ✅ Rename with `iterator = rule`; reference outer via `each.*`.
- ❌ `dynamic` over `toset([...])` of maps/objects — undefined set ordering makes plan diffs non-deterministic. ✅ Iterate a map keyed by a stable field.
- ✅ Prefer separate rule resources (`aws_vpc_security_group_ingress_rule`, provider v5+) over inline blocks + dynamics — rule changes update in place instead of recreating the parent, and each rule gets native `for_each`.

## Named anti-patterns

| Anti-pattern | Symptom | Fix |
|---|---|---|
| **God module** | one module creates VPC + EC2 + RDS + S3 + IAM + monitoring | split by single responsibility; wire via outputs |
| **Hardcoded env assumptions** | `instance_type = "m5.large"`, `Environment = "production"`, region pins inside a reusable module | everything configurable via typed variables; env values live in compositions |
| **Thin wrapper** | module only passes variables through to one upstream module, adds loose `map(any)` re-exports and no contract | consume the upstream module directly, or make the wrapper earn its layer (typed contract, enforced defaults) |
| **Envs via iteration in one root** | `for_each = toset(["dev","staging","prod"])` in a root module | separate root modules per env — separate state, contained blast radius |
| **remote_state as glue** | `terraform_remote_state` inside one team's stack, or >2 chained reads | module outputs; cloud data sources; reserve remote state for real ownership boundaries |
| **Env policy in primitives** | prod-only allowlists baked into a resource module where callers can't override | policy at composition/policy-as-code layer |

Also flag: missing `description` on inputs/outputs; `this` naming reused for multiple resources of one type (reserve for genuine singletons); `ignore_changes = all` (hides all drift — narrow to specific attributes with a comment naming the external writer); provisioners where `user_data`/cloud-init or `terraform_data` + `triggers_replace` (1.4+) fits.

## Runtime choice (Terraform vs OpenTofu)

HCL is identical; the choice affects commands, CI invocations, docs. Inference signals in an existing repo: `required_version` comments, CI invoking `terraform` vs `tofu`, lock-file provenance. ❌ `.terraform/` directory is shared by both — not a signal. If mixed/unknown: ask, or show both command forms. Document the runtime + floor in the module README requirements table.

## Locals for deletion ordering (from code-patterns)

A local that references an optional resource with `try(optional_assoc[0].vpc_id, aws_vpc.this.id)` creates an implicit dependency forcing consumers (subnets) to be destroyed before the association — the standard fix for "delete order wrong on optional resources" (VPC secondary CIDRs being the canonical case).
