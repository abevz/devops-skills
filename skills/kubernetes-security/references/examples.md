# Bad → good security examples

Concrete before/after pairs for the highest-impact findings, with the exploit path each fix
closes. Adapt to the user's manifests; don't paste verbatim.

## Unhardened pod (the default is the vulnerability)

❌ Bad — every field here is just "what you get if you don't ask":

```yaml
spec:
  containers:
    - name: app
      image: registry.example.com/app:latest
      # runs as root, all capabilities, writable rootfs,
      # default ServiceAccount token auto-mounted
```

Exploit path: an RCE in the app runs as root with a writable filesystem and a mounted
ServiceAccount token — the attacker persists in the container and starts talking to the API
server with whatever RBAC the default SA happens to have.

✅ Good:

```yaml
spec:
  automountServiceAccountToken: false      # pod doesn't call the API server
  securityContext:
    runAsNonRoot: true
    runAsUser: 10001
    seccompProfile: {type: RuntimeDefault}
  containers:
    - name: app
      image: registry.example.com/app@sha256:<digest>
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop: [ALL]
```

## hostPath to the container runtime socket (node takeover)

❌ Bad:

```yaml
volumes:
  - name: docker
    hostPath: {path: /var/run/docker.sock}
```

Exploit path: anyone who compromises this container controls the container runtime — they can
start a privileged container on the node, mount the host filesystem, and own every workload on
that node. `hostPID`/`hostNetwork`/`privileged: true` are the same severity class.

✅ Good: don't. If the workload genuinely needs runtime/node introspection (monitoring agents,
CI builders), isolate it: dedicated node pool, dedicated namespace with Pod Security Admission
`privileged` scoped to only that namespace, and no other workloads scheduled beside it. For
image builds specifically, prefer rootless builders (BuildKit rootless, kaniko) over socket
mounts.

## RBAC wildcards

❌ Bad:

```yaml
kind: ClusterRole
rules:
  - apiGroups: ["*"]
    resources: ["*"]
    verbs: ["*"]
```

Exploit path: any pod bound to this role is one compromise away from cluster-admin — read every
Secret in every namespace, create privileged pods, rewrite RBAC.

✅ Good — enumerate what's actually used, scope to a namespace with a `Role` where possible:

```yaml
kind: Role
metadata:
  namespace: app-prod
rules:
  - apiGroups: [""]
    resources: [configmaps]
    verbs: [get, list, watch]
```

Special attention to `secrets` + `get/list` (credential harvesting), `pods/exec` (lateral
movement), and `create` on `pods` or `deployments` (privilege escalation via a privileged pod
spec).

## No default-deny NetworkPolicy

❌ Bad: namespace has no NetworkPolicy at all — every pod can reach every other pod in the
cluster, so one compromised frontend pod can port-scan and attack databases in other namespaces.

✅ Good — default-deny both directions, then allow only what's needed:

```yaml
kind: NetworkPolicy
metadata:
  name: default-deny
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
---
kind: NetworkPolicy
metadata:
  name: allow-api-to-db
spec:
  podSelector:
    matchLabels: {app: db}
  policyTypes: [Ingress]
  ingress:
    - from:
        - podSelector:
            matchLabels: {app: api}
      ports:
        - {port: 5432, protocol: TCP}
```

Remember egress: default-deny egress (plus an explicit DNS allowance to kube-dns) is what stops
a compromised pod from exfiltrating data or pulling attacker tooling.

## Secrets as env vars

❌ Bad: `envFrom: secretRef` / `valueFrom: secretKeyRef` into env for high-value credentials —
env vars leak via `kubectl describe pod` (RBAC'd, but broad), crash dumps, child processes, and
frequently into application logs that print the environment on startup.

✅ Good: mount as a file (`volumes: secret` + `readOnly: true`), point the app at the path, and
rotate without a restart where the app supports re-reading. Env vars are acceptable for
low-value config; not for database credentials or signing keys.
