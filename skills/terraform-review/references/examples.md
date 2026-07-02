# Bad → good Terraform examples

Concrete before/after pairs mapped to the risk categories in `SKILL.md`. Adapt to the user's
code; don't paste verbatim.

## Secrets: hardcoded credential (risk: secrets)

❌ Bad:

```hcl
provider "postgresql" {
  host     = "prod-db.internal"
  password = "s3cr3t-changeme"   # in git history forever, and in state
}
```

✅ Good:

```hcl
variable "db_password" {
  type      = string
  sensitive = true
}

provider "postgresql" {
  host     = var.db_host
  password = var.db_password   # injected from a secrets manager / CI OIDC-scoped store
}
```

Note the limits honestly: `sensitive = true` prevents the value from appearing in plan output,
but it is **still stored in plaintext in the state file** — state encryption and state-access
RBAC are part of the fix, not an optional extra. Where the provider supports it, prefer
write-only/ephemeral arguments or data sources that fetch the secret at apply time instead of
passing it through a variable at all.

## Identity churn: for_each keyed on a mutable value (risk: identity churn)

❌ Bad — reordering or renaming one entry re-addresses (destroys/recreates) the others:

```hcl
resource "aws_iam_user" "team" {
  count = length(var.usernames)
  name  = var.usernames[count.index]
}
```

Removing `var.usernames[0]` shifts every subsequent index: `plan` shows a cascade of `-/+`
replacements for users that "didn't change."

✅ Good:

```hcl
resource "aws_iam_user" "team" {
  for_each = toset(var.usernames)
  name     = each.value
}
```

Each resource is addressed by its own stable key; adding/removing one user touches one resource.
If the change has already shipped with `count`, migrating requires `moved` blocks (or
`terraform state mv`) — flag that as part of the fix, not an afterthought.

## Destroy risk: stateful resource without lifecycle protection (risk: blast radius / state)

❌ Bad — one bad refactor away from `plan` proposing to destroy the production database:

```hcl
resource "aws_db_instance" "main" {
  identifier = "prod-main"
  engine     = "postgres"
  # ...
}
```

✅ Good:

```hcl
resource "aws_db_instance" "main" {
  identifier          = "prod-main"
  engine              = "postgres"
  deletion_protection = true          # provider-level guard

  lifecycle {
    prevent_destroy = true            # terraform-level guard: plan fails instead of destroying
  }
}
```

Belt and suspenders on purpose: `prevent_destroy` stops Terraform, `deletion_protection` stops
anything (console, API, a colleague with different state). The same pattern applies to volumes,
buckets with data, and KMS keys.

## Provider constraints: unpinned or over-pinned (risk: CI drift)

❌ Bad:

```hcl
terraform {
  required_providers {
    aws = { source = "hashicorp/aws" }          # any version — plan differs machine to machine
  }
}
```

Also bad: `version = "= 5.31.0"` everywhere — blocks security patches and forces lockstep
upgrades of every module at once.

✅ Good:

```hcl
terraform {
  required_version = ">= 1.6"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.31"     # patch/minor updates allowed, major pinned
    }
  }
}
```

Plus a committed `.terraform.lock.hcl` so CI and every engineer resolve identical provider
builds; upgrades happen as explicit, reviewed `terraform init -upgrade` commits.

## ignore_changes masking real drift (risk: state corruption / CI drift)

❌ Bad:

```hcl
lifecycle {
  ignore_changes = all
}
```

The resource's real configuration now silently diverges from code forever; the next person to
remove the block gets a terrifying plan with no idea what's intentional.

✅ Good — ignore only the specific attribute mutated by an external controller, with a comment
saying who mutates it:

```hcl
lifecycle {
  # desired_count is managed by application autoscaling, not Terraform
  ignore_changes = [desired_count]
}
```
