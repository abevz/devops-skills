# Optional local validation tooling

Optional, read-only tools the user may already have installed. Suggest them; do not install them
or run them without the user confirming the tool is present and wanting it invoked.

| Tool | Purpose | Example invocation |
|---|---|---|
| `helm lint` | Built-in chart lint (required values, template syntax) | `helm lint ./chart` |
| `helm template` | Renders the chart locally without touching a cluster, for piping into other tools | `helm template ./chart -f values-prod.yaml` |
| `kubeconform` | Validates rendered output against the Kubernetes OpenAPI schema | `helm template ./chart \| kubeconform -summary -` |
| `kube-score` | Reliability/security best-practice scoring on rendered output | `helm template ./chart \| kube-score score -` |
| `polaris` | Policy-based audit of rendered workload configuration | `helm template ./chart \| polaris audit --audit-path -` |
| `pluto` | Detects deprecated/removed API versions in rendered output | `helm template ./chart \| pluto detect -` |

Always render with `helm template` (or `--dry-run`) rather than `helm install`/`upgrade` when
piping into these tools — this keeps the review fully local and cluster-safe.
