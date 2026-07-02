# Argo CD repositories, credentials, and plugins

Self-authored reference (no third-party source). Connecting repos securely and extending
rendering. Propose config; never inline credentials or echo secret values.

## Repository connection (declarative)

Repos are k8s Secrets labeled for Argo CD — declarative, GitOps-managed (the Secret's *values*
come from a secrets mechanism, not committed plaintext):

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: repo-team-a
  namespace: argocd
  labels: {argocd.argoproj.io/secret-type: repository}
stringData:
  url: https://github.com/org/team-a
  # + credentials per type below
```

| Credential type | Fields | Note |
|---|---|---|
| HTTPS token | `username`, `password` (PAT) | Simple; token rotation needed |
| SSH | `sshPrivateKey` | Deploy key per repo; read-only key preferred |
| GitHub App | `githubAppID`, `githubAppInstallationID`, `githubAppPrivateKey` | Best for org-wide: scoped, higher rate limits, no per-user PAT |

- **repo-creds templates**: a Secret labeled `repo-creds` with a `url` prefix applies to all
  repos under that prefix — connect one org credential, not one Secret per repo.
- SSH vs HTTPS are *different repo entries* — a repo added as HTTPS won't match SSH creds (common
  "authentication required" cause, cross-ref `argocd-debug`).
- Never commit these Secrets with real values — generate them via External Secrets / SOPS /
  Sealed Secrets (see `secrets-management`); the ExternalSecret referencing the PAT is what's in
  git.

## Helm OCI and chart repositories

```yaml
stringData:
  url: registry.example.com               # OCI registry
  type: helm
  enableOCI: "true"
  username: ...
  password: ...
```

Helm dependency repos must be reachable by repo-server; private chart deps need their creds
registered too, or `helm dependency build` fails during render.

## Clusters (managed destinations)

External clusters are also labeled Secrets (`argocd.argoproj.io/secret-type: cluster`) holding
the API endpoint + credentials (bearer token / client cert / exec plugin for cloud IAM). Scope
the ServiceAccount Argo CD uses on each managed cluster to least privilege — Argo CD's cluster
credential is a high-value target; a broad one turns an Argo CD compromise into every-cluster
compromise.

## Config Management Plugins (CMP)

When rendering needs tooling Argo CD doesn't bundle (jsonnet, cdk8s, cue, sops decryption,
custom templating), run a CMP as a **sidecar to repo-server**:

```yaml
# a ConfigMap defines the plugin's generate command; mounted as a repo-server sidecar
# Application references it via spec.source.plugin.name
```

- CMP sidecars share repo-server's access — a plugin that shells out is a supply-chain surface;
  pin the sidecar image and review what it executes.
- Legacy in-repo-server plugins (via `argocd-cm`) are deprecated in favor of sidecar CMPs.

## Secret injection at render (Vault plugin / avoid it)

- `argocd-vault-plugin` (a CMP) substitutes `<path:secret#key>` placeholders from Vault at render
  time — keeps secrets out of git, but Argo CD now sees plaintext secrets in rendered manifests
  (and its cache). Weigh against External Secrets Operator, which keeps the secret flow *outside*
  Argo CD entirely (Argo CD just applies an `ExternalSecret` CR). Prefer ESO unless there's a
  reason the value must be resolved at render.
- Whichever: the resulting `Secret` is what ends up in-cluster — apply the mounted-file vs env
  and least-access rules from `kubernetes-security`.

## Verify (read-only)

```bash
argocd repo list
argocd repo get https://github.com/org/team-a
kubectl get secrets -n argocd -l argocd.argoproj.io/secret-type=repository
argocd cluster list
```
