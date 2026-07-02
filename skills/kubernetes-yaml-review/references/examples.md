# Bad → good manifest examples

Concrete before/after pairs for the most common findings. Use these to recognize the pattern and
to show the user what the fix looks like — adapt names/values to their manifest, don't paste
verbatim.

## Missing probes, resources, and pinned image

❌ Bad — schedules anywhere, no health gating, floating tag:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 2
  selector:
    matchLabels: {app: api}
  template:
    metadata:
      labels: {app: api}
    spec:
      containers:
        - name: api
          image: registry.example.com/api:latest
          ports: [{containerPort: 8080}]
```

✅ Good:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 2
  selector:
    matchLabels: {app: api}
  template:
    metadata:
      labels: {app: api}
    spec:
      containers:
        - name: api
          image: registry.example.com/api:1.42.3   # or @sha256:<digest>
          ports: [{containerPort: 8080}]
          resources:
            requests: {cpu: 100m, memory: 128Mi}
            limits: {memory: 256Mi}
          readinessProbe:
            httpGet: {path: /healthz/ready, port: 8080}
            periodSeconds: 5
          livenessProbe:
            httpGet: {path: /healthz/live, port: 8080}
            initialDelaySeconds: 10
            periodSeconds: 10
```

Why: no readiness probe means traffic hits pods before they can serve; no requests means the
scheduler packs blindly and QoS is BestEffort (first evicted); `:latest` makes rollbacks and
incident forensics impossible.

## Selector / label mismatch (silent orphaning)

❌ Bad — Service selects nothing because labels drifted:

```yaml
kind: Service
spec:
  selector: {app: api-server}     # pods are labeled app: api
```

✅ Good — selector matches the pod template labels exactly, and both come from one place (Helm
helper or Kustomize commonLabels) so they can't drift independently.

## Overly aggressive liveness probe (restart storm)

❌ Bad:

```yaml
livenessProbe:
  httpGet: {path: /healthz, port: 8080}
  initialDelaySeconds: 1
  periodSeconds: 2
  timeoutSeconds: 1
  failureThreshold: 1
```

One slow GC pause or dependency hiccup kills the pod; under load this cascades into a
namespace-wide restart storm.

✅ Good: `failureThreshold: 3`+, `timeoutSeconds` ≥ 2–5, and the liveness endpoint must not
check downstream dependencies (that's readiness's job — liveness answers "is this process
wedged," not "is the world healthy").

## Everything on one node

❌ Bad — 3 replicas, no spread constraints: a single node failure takes all replicas down.

✅ Good:

```yaml
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: ScheduleAnyway
    labelSelector:
      matchLabels: {app: api}
```

Plus a PodDisruptionBudget (`minAvailable: 1` or `maxUnavailable: 1`) so voluntary drains during
node upgrades can't evict everything at once.

## Hardcoded config and secrets in env

❌ Bad:

```yaml
env:
  - name: DB_PASSWORD
    value: "s3cr3t-changeme"
  - name: DB_HOST
    value: "prod-db.internal"
```

✅ Good: non-secret config via `ConfigMap` (per-environment), secrets via `secretKeyRef` backed
by External Secrets / Sealed Secrets / SOPS — never a literal in the manifest, which ends up in
git history and `kubectl describe` output.
