# Optional local validation and scanning tooling

Optional, read-only tools the user may already have installed. Suggest them; do not install them
or run them without the user confirming the tool is present and wanting it invoked.

| Tool | Purpose | Example invocation |
|---|---|---|
| `terraform validate` / `tofu validate` | Built-in syntax/internal-consistency check | `terraform validate` |
| `terraform fmt -check` | Formatting check, no state access | `terraform fmt -check -recursive` |
| `tflint` | Provider-aware linting (deprecated syntax, provider best practices) | `tflint --recursive` |
| `checkov` | IaC security/compliance scanner (Terraform, CloudFormation, etc.) | `checkov -d .` |
| `trivy` | Vulnerability and misconfiguration scanning, including `trivy config` for Terraform | `trivy config .` |
| `gitleaks` | Scans for committed secrets in `.tf`/`.tfvars` files | `gitleaks detect --source .` |

`terraform plan` is read-only against the backend but does contact the configured provider/state
— only run it with explicit user confirmation, and never follow it with `apply`/`destroy` as part
of this skill.
