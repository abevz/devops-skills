# Validation Pipeline, Policy Engines & Kustomize Reference

Distilled from LukasNiessen/kubernetes-skill (`references/validation-and-policy.md`, `kustomize-patterns.md`) during third-party review.

## Validation pipeline order

Each layer catches a different error class — order matters:

| Stage | Tool | Catches | Misses |
|---|---|---|---|
| 1. Schema | `kubeconform -strict -kubernetes-version <target>` | Structural errors, unknown/misspelled fields, removed apiVersions | Org rules, cluster state |
| 2. Lint/render | `helm lint` / `helm template` / `kustomize build` | Template and overlay breakage | Everything semantic |
| 3. Policy | Kyverno CLI (`kyverno apply policies/ --resource rendered.yaml`) / Gatekeeper / `polaris audit` | Org rule violations (limits, labels, registries, privilege) | Webhook rejections, quotas |
| 4. Cluster dry-run | `kubectl apply --dry-run=server` (non-persisting) | Admission webhooks, quota violations, naming conflicts | Nothing structural |

Key flags/pitfalls:
- `kubeconform` without `-strict` lets unknown fields pass silently; always pin `-kubernetes-version` to the target cluster.
- CRDs are **silently skipped** without a schema location: add `-schema-location default -schema-location 'https://raw.githubusercontent.com/datreeio/CRDs-catalog/main/{{.Group}}/{{.ResourceKind}}_{{.ResourceAPIVersion}}.json'`.
- Bare `--dry-run` is deprecated (defaults to client); be explicit. `--dry-run=client` is syntax-only — it cannot catch webhook or quota failures, so a pipeline without server dry-run has a blind spot.
- Pipe rendered output, never raw templates: `helm template rel ./chart -f values-prod.yaml | kubeconform ...`; `kustomize build overlays/production | kubeconform ...`.
- Polaris for score-based gating: `polaris audit --audit-path manifests/ --set-exit-code-on-danger`.

GitHub Actions shape: checkout → render (`helm template`/`kustomize build`) → kubeconform → `kyverno apply policies/ --resource rendered.yaml` (kyverno/action-install-cli).

## Kyverno

YAML-native; policies are cluster resources. `ClusterPolicy` = cluster-wide, `Policy` = namespaced. Rule types: validate, mutate, generate, verifyImages.

- ❌ `validationFailureAction: Audit` in production — logs violations without blocking. ✅ `Enforce`.
- `background: true` also evaluates existing resources, not just new admissions.
- Match controller kinds (Deployment/StatefulSet/DaemonSet) or rely on Kyverno rule auto-generation — matching `Pod` alone misses nothing at runtime (pods are created) but label/metadata policies on `Pod` only miss the controller objects.

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata: {name: require-resource-limits}
spec:
  validationFailureAction: Enforce
  background: true
  rules:
    - name: check-limits
      match: {any: [{resources: {kinds: [Pod]}}]}
      validate:
        message: "All containers must have memory and cpu limits."
        pattern:
          spec:
            containers:
              - resources: {limits: {memory: "?*", cpu: "?*"}}
```

Same pattern for required labels (`app.kubernetes.io/name: "?*"`, `app.kubernetes.io/version: "?*"` on Deployment/StatefulSet/DaemonSet) and registry allowlisting (`containers: [{image: "registry.example.com/*"}]` — include `initContainers` too).

## OPA Gatekeeper

Two-object model: `ConstraintTemplate` (Rego logic, defines a CRD kind) + `Constraint` instance (match criteria + parameters).

```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata: {name: k8sdisallowprivileged}
spec:
  crd: {spec: {names: {kind: K8sDisallowPrivileged}}}
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8sdisallowprivileged
        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          container.securityContext.privileged == true
          msg := sprintf("Container '%v' must not be privileged", [container.name])
        }
        violation[{"msg": msg}] {
          container := input.review.object.spec.initContainers[_]   # REQUIRED — init containers bypass otherwise
          container.securityContext.privileged == true
          msg := sprintf("Init container '%v' must not be privileged", [container.name])
        }
---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sDisallowPrivileged
metadata: {name: no-privileged-containers}
spec:
  match: {kinds: [{apiGroups: [""], kinds: [Pod]}]}
```

Review points: every container-level Rego rule must cover `containers` AND `initContainers` (and ephemeralContainers where relevant); test policies on both create and update — some behave differently per operation.

## Kustomize overlay structure

```
app/
  base/          kustomization.yaml + deployment.yaml + service.yaml
  overlays/
    dev/         kustomization.yaml + replica-patch.yaml
    staging/
    production/  kustomization.yaml + resource-patch.yaml + hpa.yaml
  components/
    monitoring/  kustomization.yaml (kind: Component) + servicemonitor.yaml
```

Every kustomization.yaml declares `apiVersion: kustomize.config.k8s.io/v1beta1`, `kind: Kustomization`, `resources`. Overlays reference `../../base` (the **directory**, not files). Components use `apiVersion: kustomize.config.k8s.io/v1alpha1, kind: Component` and are included via `components:` — reusable cross-cutting patches (e.g. add scrape annotations + ServiceMonitor) shared across overlays.

### Patch selection

| Need | Mechanism |
|---|---|
| Merge/override fields into an existing structure | Strategic merge patch (`patches: [{path: file.yaml}]`) — patch file must carry matching `kind` + `metadata.name` |
| Add/remove/replace at exact path, array element manipulation | JSON patch with `target: {kind, name}`; `/-` appends, explicit index targets a position |

### Pitfalls (patch ordering / base drift / generators)

- ❌ `bases:` — deprecated; use `resources:`.
- ❌ `commonLabels` on resources with existing deployed selectors — it injects into `spec.selector.matchLabels`, which is immutable → upgrade fails. Use `commonAnnotations` or label only templates.
- ❌ Strategic merge patch missing `metadata.name` — silently matches nothing. Same for JSON-patch `target` with wrong group/version/kind.
- ❌ Overlay generator without `behavior: merge` — creates a *second* ConfigMap instead of extending the base one (base drift: overlay and base diverge silently).
- ConfigMap/Secret generators append a content hash to the name (this is what triggers rolling updates on config change) — any hardcoded reference to the un-hashed name in a non-managed manifest won't resolve.
- JSON patch array index errors: appending with a wrong explicit index inserts at the wrong position; prefer `/-`.
- Image pinning: `images:` transformer with `digest: sha256:...` over `newTag` for production.

### Kustomize vs Helm decision

| Scenario | Use |
|---|---|
| Env overlays on static manifests, 2–3 envs | Kustomize |
| Heavy parameterization, many knobs, loops/conditionals | Helm |
| Consuming third-party charts | Helm (Kustomize cannot) |
| Shared template library | Helm library charts |
| CRDs / operator-managed resources | Either |

## Read-only verification

```bash
kustomize build overlays/production                          # render — fails loudly on structural errors
kustomize build overlays/production | kubeconform -kubernetes-version 1.29.0 -strict
helm template rel ./chart -f values-prod.yaml | kubeconform -kubernetes-version 1.29.0 -strict
kyverno apply policies/ --resource rendered.yaml             # offline policy evaluation
polaris audit --audit-path manifests/ --format score
kubectl apply -f rendered.yaml --dry-run=server              # non-persisting; full admission chain
```
