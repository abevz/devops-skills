# Optional local security scanning tooling

Optional, read-only tools the user may already have installed. Suggest them; do not install them
or run them without the user confirming the tool is present and wanting it invoked.

| Tool | Purpose | Example invocation |
|---|---|---|
| `trivy` | Scans container images, filesystems, and IaC configs for known CVEs and misconfigurations | `trivy image <image>` / `trivy config manifests/` |
| `gitleaks` | Scans a repo/diff for committed secrets (API keys, tokens, private keys) | `gitleaks detect --source .` |
| `kube-linter` | Flags insecure defaults (missing securityContext, privilege escalation) | `kube-linter lint manifests/` |
| `polaris` | Policy-based audit including a security-focused ruleset | `polaris audit --audit-path manifests/` |
| `conftest` | Runs OPA/Rego policies against manifests (custom org security policy as code) | `conftest test manifests/ -p policy/` |

Tool output flags *possible* issues; still confirm exploitability and blast radius manually before
ranking severity, per the workflow in `SKILL.md`.
