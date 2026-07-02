# Helm Chart Patterns Reference

Distilled from LukasNiessen/kubernetes-skill (`references/helm-patterns.md`, plus the Helm sections of `api-drift.md`) during third-party review.

## Chart.yaml contract

```yaml
apiVersion: v2            # mandatory for Helm 3
name: my-app
version: 0.1.0            # chart SemVer — MUST bump on every chart change (stale repo cache otherwise)
appVersion: "1.0.0"       # app release, tracked independently of chart version
type: application         # or "library"
description: "..."
```

- ❌ Chart change without `version` bump — Helm repos serve the stale cached version.
- Dependencies: declare sub-charts under `dependencies` in Chart.yaml; `helm dependency update` generates `Chart.lock`; commit **both**. Use `condition`/`tags` to make sub-charts optional.

## values.yaml design

- Group by resource type (image, securityContext, resources, probes, ingress, serviceAccount); every section documented with `# --` comments (helm-docs convention).
- Safe-by-default: PSS-restricted securityContext in defaults (`runAsNonRoot: true`, `runAsUser: 65534`, `allowPrivilegeEscalation: false`, `readOnlyRootFilesystem: true`, `capabilities.drop: [ALL]`), resources set even if minimal.
- `image.tag: ""` defaulting to `.Chart.AppVersion` in the template — but the template must supply that default: `{{ .Values.image.tag | default .Chart.AppVersion }}`. ❌ No default → rendered `repository:` with trailing colon.
- Standard toggles: `ingress.enabled: false`, `serviceAccount.create: true` + `serviceAccount.name: ""`.
- Mandatory values with no sane default: guard with `required "message" .Values.x` — otherwise the chart installs with nils and pods crash at runtime.

## Template safety

| Rule | Failure mode if violated |
|---|---|
| `{{- ... -}}` whitespace trimming | Blank lines break multi-document YAML |
| `| nindent N` after every `include`/`toYaml` | Nested objects render at column 0 → parse failure |
| `{{ .Values.foo | quote }}` for strings | Numeric/special-char values break YAML typing |
| Labels via named templates (`include "chart.labels"`), never inlined | Selector/label mismatch when overridden |
| `{{ .Values.replicas | default 3 }}` or `if` guard | Nil render when value undefined |
| Conditional resources wrapped in `{{- if .Values.x.enabled }}` | Resources rendered when disabled |

Required `_helpers.tpl` set (all `trunc 63 | trimSuffix "-"` — DNS label limit):

```yaml
{{- define "mychart.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}

{{- define "mychart.labels" -}}
helm.sh/chart: {{ include "mychart.chart" . }}
app.kubernetes.io/name: {{ include "mychart.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{- define "mychart.selectorLabels" -}}
app.kubernetes.io/name: {{ include "mychart.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}

{{- define "mychart.serviceAccountName" -}}
{{- if .Values.serviceAccount.create }}
{{- default (include "mychart.fullname" .) .Values.serviceAccount.name }}
{{- else }}
{{- default "default" .Values.serviceAccount.name }}
{{- end }}
{{- end }}
```

`selectorLabels` is deliberately the two-label subset — **never** include `app.kubernetes.io/version` in a selector (selectors are immutable; version there breaks every upgrade).

Deployment template shape:

```yaml
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "mychart.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "mychart.selectorLabels" . | nindent 8 }}
    spec:
      serviceAccountName: {{ include "mychart.serviceAccountName" . }}
      securityContext:
        {{- toYaml .Values.securityContext | nindent 8 }}
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          livenessProbe:
            httpGet:
              path: {{ .Values.probes.liveness.path }}
              port: {{ .Values.probes.liveness.port }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

## API-version handling in templates

Never hardcode beta apiVersions. Branch on cluster capabilities when supporting multiple versions:

```yaml
{{- if .Capabilities.APIVersions.Has "networking.k8s.io/v1" }}
apiVersion: networking.k8s.io/v1
{{- else }}
apiVersion: networking.k8s.io/v1beta1
{{- end }}
```

Symptom: `helm template` renders fine but apply fails — templating validates syntax, not apiVersions against the cluster.

## Upgrade behavior

- Selector labels immutable → any change to `selectorLabels` output (renaming release, changing nameOverride into the selector) forces delete/recreate of Deployments.
- Chart `version` bump signals change; `appVersion` bump alone with cached chart version deploys nothing new from a repo.
- `Chart.lock` drift: uncommitted lock means CI resolves different sub-chart versions than the author tested.

## Verification order (dev and CI, all cluster-safe)

1. `helm lint ./chart` — syntax/structure.
2. `helm template release-name ./chart -f values-prod.yaml` — render per environment values file.
3. `... | kubeconform -kubernetes-version 1.29.0 -strict` — schema-validate rendered output against the target cluster version.
4. `helm test release-name` — post-install in-cluster test pods (only on an already-installed release, with user confirmation).

## Review checklist (source's mistake list, condensed)

- [ ] `{{-` trimming everywhere; no blank-line artifacts in `helm template` output
- [ ] `nindent` on every block include/toYaml
- [ ] `quote` on templated string values
- [ ] No hardcoded label sets; helpers used for labels AND selectors
- [ ] `image.tag` has `default .Chart.AppVersion`
- [ ] Chart `version` bumped in the diff
- [ ] `required` guards on mandatory values
- [ ] `Capabilities.APIVersions` branch (or current stable apiVersions only)
- [ ] Chart.yaml + Chart.lock both committed
