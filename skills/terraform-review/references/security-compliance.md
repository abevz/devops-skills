# Security & compliance — secrets, state encryption, policy-as-code

Distilled from antonbabenko/terraform-skill (`references/security-compliance.md`, secrets floors cross-checked against `references/code-patterns.md`) during third-party review.
Use when reviewing secret handling, state backend hardening, credentials, or compliance claims.

## Secrets: what actually lands in state

The core review question is never "is it marked sensitive" but "does the value persist in the state file".

| Mechanism | Floor (Terraform) | In state? | Verdict |
|---|---|---|---|
| `variable` with `sensitive = true` | 0.14+ | **YES** — masks display only | ❌ not a secrets solution |
| `random_password.result` → resource arg | any | **YES** (both resource and random state) | ❌ |
| `data "aws_secretsmanager_secret_version"` | any | **YES** — `secret_string` re-read into state on every refresh | ⚠️ avoids hardcoding, does not exclude from state |
| Write-only args (`password_wo` + `password_wo_version`) | 1.11+ (AWS provider ≥5.71) | resource argument NO | ✅ — but see pairing trap below |
| `ephemeral` resources/values | 1.10+ (OpenTofu: shipped later — verify release notes; e.g. `ephemeral "random_password"` needs random provider ≥3.7.0) | NO — never in state or plan | ✅ for short-lived credentials |
| `manage_master_user_password = true` (RDS) | provider feature | NO — AWS generates and stores in Secrets Manager | ✅ recommended default for RDS |
| CI-injected env var (`TF_VAR_*` set in pipeline only) | any | value flows through plan; not hardcoded/committed | ✅ pre-1.10 fallback |

Traps to flag:
- ❌ `password_wo = data.aws_secretsmanager_secret_version.x.secret_string` — the write-only arg stays out of state but the **data source still persists the secret on refresh**. True exclusion: `ephemeral` (1.10+), `manage_master_user_password`, or CI env var.
- ❌ `nonsensitive()` used to silence a sensitive-value warning — launders the secret into plan output/CI artifacts. Legitimate only for provably non-secret derived values.
- ❌ Plaintext defaults in `variable` blocks or committed `.tfvars` "for demo convenience". `.gitignore` must cover `*.tfvars`, `.env`, `secrets/`.
- ❌ Secrets echoed via `provisioner`/`local-exec` stdout — lands in CI logs; `sensitive` doesn't redact process output.
- ❌ Outputs exposing connection strings/credentials, even with `sensitive = true` (still in state, still retrievable via `output -json`).
- ❌ `sensitive = true` recommended as the fix on a 1.11+ runtime where `*_wo` exists; conversely, ❌ emitting `*_wo`/`ephemeral` without checking the runtime floor.
- Verification step after a secrets refactor: `terraform show | grep -i password` must come back empty.
- Manually-managed secrets: create the `aws_secretsmanager_secret` container in Terraform, populate `secret_string` out-of-band (console/CLI/write-only) — never via `random_password` stored in state.
- After external rotation: `terraform apply -refresh-only` reconciles state (state mutation — confirm first); it does not rotate anything.

Recommended RDS shape (nothing secret in state):

```hcl
resource "aws_kms_key" "db" {
  description             = "CMK for RDS-managed master password"
  enable_key_rotation     = true
  deletion_window_in_days = 30
}

resource "aws_db_instance" "this" {
  # Option 1 (recommended): AWS generates + stores password in Secrets Manager
  manage_master_user_password   = true
  master_user_secret_kms_key_id = aws_kms_key.db.arn
  lifecycle { ignore_changes = [master_user_secret] }

  # Option 2 (Terraform 1.11+, AWS provider >= 5.71): write-only argument
  # password_wo         = ephemeral.random_password.db.result   # ephemeral: 1.10+
  # password_wo_version = 1
}
```

## Provider credentials: OIDC vs static keys

- ✅ Keyless OIDC federation for all three clouds (AWS OIDC roles, Azure federated credentials, GCP Workload Identity Federation). Static keys only when OIDC is genuinely unavailable (non-OIDC CI, some self-hosted runners).
- ❌ Long-lived cloud keys stored as CI secrets when the platform supports OIDC — flag as a finding.
- Trust-policy correctness (aud/sub pinning table and wildcard-`sub` traps) lives in cicd-review `references/terraform-pipelines.md` — apply it whenever reviewing the IAM side.
- ✅ Per-environment IAM roles scoped to their own state path (`prod/*` only) — workspace separation is not IAM separation.

## Encryption at rest for state backends

`encrypt = true` on the S3 backend alone = SSE-S3 (AWS-managed AES-256, no per-request CloudTrail audit). State usually holds secrets, so:

```hcl
terraform {
  backend "s3" {
    bucket       = "my-terraform-state"
    key          = "prod/vpc/terraform.tfstate"
    region       = "us-east-1"
    encrypt      = true                                        # SSE on PUT
    kms_key_id   = "arn:aws:kms:us-east-1:ACCOUNT:key/KEY-ID"  # customer-managed CMK
    use_lockfile = true                                        # Terraform 1.10+/OpenTofu 1.10+ native lock
  }
}
```

State bucket hardening checklist (each missing item is a finding):
- [ ] Versioning enabled (restore path for corruption/deletion)
- [ ] SSE-KMS with customer-managed CMK, `enable_key_rotation = true`, `bucket_key_enabled = true` (controls KMS request cost)
- [ ] `aws_s3_bucket_public_access_block` — all four flags true
- [ ] Bucket policy `Deny` on `"aws:SecureTransport": "false"` (TLS-only regardless of other grants)
- [ ] IAM: `s3:ListBucket` on the **bucket ARN**, object actions (`GetObject`/`PutObject`/`DeleteObject`/`GetObjectVersion`) on `/*` — mismatched pairing silently no-ops; `DeleteObject`+`GetObjectVersion` needed with versioning on
- [ ] Access logging / CloudTrail data events on the state prefix (who read/modified state)
- [ ] `prevent_destroy` on the bucket

Cross-cloud: Azure Storage and GCS encrypt at rest by default (optional CMK/CMEK); AWS is the one requiring explicit config. GCS backend CMEK goes on the **bucket** (`default_kms_key_name`) — the backend's `encryption_key` attr is CSEK (raw base64 AES key), not a KMS resource name. OpenTofu 1.7+ additionally offers native client-side **state encryption** (no Terraform equivalent) — an option when backend-level encryption is insufficient.

Regulated workloads (HIPAA/PCI/FedRAMP): SSE-S3/`AES256` is usually insufficient — require `aws:kms` with a rotated customer-managed CMK, for the state bucket and for data resources alike.

## Policy-as-code: pipeline placement

| Tool | Input | Where it runs | Notes |
|---|---|---|---|
| trivy config / checkov | HCL source | PR validate stage (pre-plan, fast, free) | Trivy absorbed tfsec's rules (tfsec in maintenance since 2022); `checkov -d . --framework terraform`, skip via `--skip-check CKV_AWS_23` |
| Conftest / OPA (Rego) | plan JSON | after plan, before apply gate | `terraform plan -out=tfplan && terraform show -json tfplan > tfplan.json && conftest test tfplan.json --policy policy/` |
| Sentinel | plan | Terraform Cloud/Enterprise only | built into TFC run pipeline |
| terraform-compliance (BDD) | — | ❌ archived, unmaintained — flag if proposed; use Conftest/OPA |

Placement rules:
- ✅ Static scan (trivy/checkov) on every PR; plan-JSON policy (OPA/Conftest) between plan and apply — it sees final resolved values, catching what static analysis can't.
- ❌ Pipeline "claims compliance" but has no policy stage — flag it.
- ⚠️ Plan JSON (`terraform show -json`) can contain sensitive values — restrict artifact access and retention (see terraform-pipelines.md).
- OPA plan-policy modeling note: AWS provider v4+ moved S3 encryption to the separate `aws_s3_bucket_server_side_encryption_configuration` resource — policies must join bucket ↔ encryption-config resources, not look for inline `server_side_encryption` on the bucket. Accepted algorithms: `aws:kms`, `aws:kms:dsse`, `AES256`.

## Resource-level security review flags

- ❌ SG rules with `protocol = "-1"` + `cidr_blocks = ["0.0.0.0/0"]` (worst on ingress; scope egress to needed ports too).
- ❌ Inline `ingress`/`egress` blocks on `aws_security_group` — rule edits recreate the whole SG; mixing inline + separate resources conflicts. ✅ `aws_vpc_security_group_ingress_rule`/`egress_rule` (provider v5+).
- ❌ Default VPC / default subnets for workloads — dedicated VPC with private subnets.
- ❌ IAM `Action = "*"` / `Resource = "*"` — least privilege, specific actions on specific ARNs.
- ❌ Unencrypted data stores; prefer KMS CMK over `AES256` where audit trail matters.
- Cross-cloud map: secrets — Secrets Manager / Key Vault / Secret Manager; firewalling — SG rules / NSG rules / `google_compute_firewall`; encryption at rest — AWS explicit, Azure/GCP default-on with optional CMK.

## Compliance checklist items worth keeping

- **SOC 2:** encryption at rest + in transit everywhere; least-privilege IAM; logging on all resources; MFA for privileged access (enforced at org/IdP level, not per-resource); security scanning in CI.
- **HIPAA:** PHI encrypted at rest/in transit; access logs; dedicated VPC + private subnets; backup/retention policies; audit trail for all infra changes.
- **PCI-DSS:** network segmentation (separate VPCs); no default passwords; strong algorithms; recurring scans; access control + monitoring.
- Review-stance rules:
  - ❌ Naming a framework (SOC 2/PCI/HIPAA/GDPR/FedRAMP) with no enforceable gate — no policy stage, no approval model, no evidence artifact.
  - ❌ Conflating a security control with compliance evidence — an encrypted bucket is not a retained audit artifact proving encryption.
  - ❌ Ignoring plan-JSON artifact retention/access controls, or data-residency obligations (GDPR/FedRAMP).
