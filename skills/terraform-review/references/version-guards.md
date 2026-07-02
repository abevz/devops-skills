# Version guards and operational protocols

Distilled from antonbabenko/terraform-skill (`references/quick-reference.md`) during third-party
review. Use when a finding or recommendation depends on the Terraform/OpenTofu version, or when
reviewing test/state operational practice.

## Version-gated features — check the floor before recommending

| Feature | Terraform | OpenTofu | If below the floor |
|---|---|---|---|
| Native test framework (`.tftest.hcl`, `terraform test`) | 1.6+ | 1.6+ | Use Terratest / static analysis + plan review |
| Mock providers for unit tests | 1.7+ | 1.7+ | Real-resource tests only; watch cost |
| `import` blocks (declarative import, `-generate-config-out`) | 1.5+ | 1.6+ | `terraform import` CLI (state mutation — confirm first) |
| `moved` blocks | 1.1+ | all | `terraform state mv` (state mutation — confirm first) |
| State encryption (native) | — | 1.7+ | Backend-level encryption only |

Never recommend a feature without stating its version floor — check `required_version` in the
module first. Terraform (BUSL-1.1, HashiCorp) and OpenTofu (MPL 2.0, Linux Foundation) diverge
after 1.6: encryption, mock-provider details, and provider functions differ — verify per
version, and show both `terraform` and `tofu` command forms when the user's choice is unknown.

## Stuck state lock protocol

`Error acquiring the state lock` — the lock info names who/when. In order:

1. **Verify the operation is genuinely not running** — check the host in the lock info
   (`ps aux | grep terraform` there) or the CI job it names. A running apply you unlock under
   corrupts state.
2. Only then: `terraform force-unlock <LOCK_ID>` — never `-force` flags beyond this, never
   deleting lock files directly in the backend.
3. Document the unlock (why, when, verified-not-running evidence) — repeated stuck locks point
   at CI jobs being killed mid-apply, which is the actual bug to fix (job timeouts vs apply
   duration).

Rate this as **state-corruption risk category** in review output.

## "Tests pass locally, fail in CI"

Almost always version skew: unpinned Terraform/provider versions resolving differently. Check
`required_version`, provider `version` constraints, and a committed `.terraform.lock.hcl` before
debugging the test logic itself.

## Test-cost hygiene (when reviewing test setups)

- Unit-level checks on mock providers (1.7+) instead of real resources where possible.
- Unique resource names per run (random suffix) — parallel test collisions show up as
  `ResourceAlreadyExists`.
- Integration tests gated to main branch, smallest viable instance types, TTL tags on
  everything a test creates so orphans are sweepable.
