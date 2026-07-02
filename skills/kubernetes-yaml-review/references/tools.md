# Optional local validation tooling

These are optional, read-only static-analysis tools the user may already have installed. Suggest
running them (or ask the user to run them and share output) instead of running them yourself
unless the user confirms the tool is installed and asks you to invoke it. Never suggest installing
them automatically.

| Tool | Purpose | Example invocation |
|---|---|---|
| `kubeconform` | Validates manifests against the Kubernetes OpenAPI schema for a target version | `kubeconform -summary manifests/` |
| `kube-score` | Static analysis for reliability/security best practices on rendered manifests | `kube-score score manifests/*.yaml` |
| `kube-linter` | Lints manifests/Helm charts for common misconfigurations | `kube-linter lint manifests/` |
| `polaris` | Policy-based audit of workload configuration (resources, probes, security) | `polaris audit --audit-path manifests/` |
| `pluto` | Detects use of deprecated/removed Kubernetes API versions | `pluto detect-files -d manifests/` |
| `yamllint` | General YAML syntax/style linting | `yamllint manifests/` |
| `kustomize build` | Renders a Kustomize overlay so it can be reviewed/piped into the tools above | `kustomize build overlays/prod \| kubeconform -summary -` |

These complement, not replace, the manual checklist in `SKILL.md` — tool output still needs human
(or agent) judgment about context and severity.
