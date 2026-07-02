# Optional local validation tooling

Optional, read-only tools the user may already have installed. Suggest them; do not install them
or run them without the user confirming the tool is present and wanting it invoked.

| Tool | Purpose | Example invocation |
|---|---|---|
| `kustomize build` | Renders overlays locally so drift/promotion diffs can be reviewed without a cluster | `kustomize build overlays/prod` |
| `conftest` | Runs OPA/Rego policy checks against rendered manifests pre-merge | `conftest test <(kustomize build overlays/prod) -p policy/` |
| `yamllint` | General YAML syntax/style linting across the repo | `yamllint .` |
| `gitleaks` | Scans the GitOps repo for accidentally committed secrets | `gitleaks detect --source .` |

These support the manual repo-structure review in `SKILL.md`; they don't replace checking
environment separation, promotion flow, and secrets *strategy* by hand.
