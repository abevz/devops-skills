# Cluster & Workload Hardening Reference

Distilled from LukasNiessen/kubernetes-skill (`references/insecure-workload-defaults.md`, `privilege-sprawl.md`, `network-exposure.md`, `multi-tenancy.md`, `security-hardening.md`) during third-party review.

## Pod Security Admission (PSA)

| PSS level | Use for |
|---|---|
| `restricted` | Default for all app workloads |
| `baseline` | Only when restricted is provably impossible |
| `privileged` | CNI, CSI node drivers, node agents — never apps |

Label every namespace with all three modes AND version pinning (unpinned `latest` changes behavior on cluster upgrade):

```yaml
metadata:
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest   # pin to e.g. "v1.30" for stable behavior
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: latest
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: latest
```

Migration path: `enforce: baseline` + `audit/warn: restricted`, then promote enforce.

### securityContext field placement (common review miss)

| Field | Level |
|---|---|
| `runAsUser`, `runAsGroup`, `fsGroup`, `seccompProfile` | Pod |
| `allowPrivilegeEscalation`, `readOnlyRootFilesystem`, `capabilities` | Container |
| `runAsNonRoot` | Either; pod level preferred |

Full restricted baseline per container: `runAsNonRoot: true`, `allowPrivilegeEscalation: false`, `readOnlyRootFilesystem: true`, `capabilities.drop: [ALL]`, `seccompProfile.type: RuntimeDefault`, non-zero `runAsUser`/`runAsGroup`. `readOnlyRootFilesystem` requires `emptyDir` at `/tmp` (with `sizeLimit`) or the app crashes.

- ❌ securityContext only at one level (leaves gaps: pod-level lacks caps/RO-fs; container-level misses init containers)
- ✅ Both levels; init-container deviations documented inline (e.g. `add: [CHOWN]` with drop ALL first)
- ❌ `hostNetwork`/`hostPID`/`hostIPC: true` on app workloads (`hostUsers: false` available 1.28+)
- ❌ `image: app:latest` — require registry prefix + digest or immutable tag
- ❌ `automountServiceAccountToken` left default (true) on pods that never call the API

## Framework mapping

### NSA/CISA Kubernetes Hardening Guide — key controls
| Area | Control |
|---|---|
| Pod security | PSS restricted, non-root, RO filesystem, drop ALL caps |
| Network | Default-deny NetworkPolicy per namespace; mTLS (mesh) for encryption — NetworkPolicy segments but does not encrypt |
| AuthN | `--anonymous-auth=false`, short-lived tokens, OIDC for humans |
| AuthZ | Least-privilege RBAC, no cluster-admin for workloads, audit RoleBindings |
| Audit | API server audit log at Metadata minimum, shipped off-cluster |
| Threat detection | Falco/Tetragon syscall monitoring |
| Upgrades | Within one minor of latest |

### OWASP Kubernetes Top 10
K01 insecure workload config · K02 supply chain (registry allowlist admission policy, Trivy `--severity CRITICAL,HIGH` CI gate, cosign sign/verify, syft SBOM, SLSA provenance) · K03 permissive RBAC · K04 no centralized policy (Kyverno/Gatekeeper) · K05 logging/monitoring gaps · K06 broken authn · K07 missing network segmentation · K08 secrets failures (etcd encryption) · K09 misconfigured components (CIS: apiserver `--authorization-mode=RBAC,Node`, kubelet `--read-only-port=0`, `--authorization-mode=Webhook`, etcd peer TLS) · K10 outdated components.

## RBAC privilege-escalation vectors

Verbs/resources that are equivalent to broader access than they look:

| Grant | Escalation path |
|---|---|
| `get`/`list` on `secrets` (namespace) | Read every SA token + credential in the namespace → impersonate any SA there |
| `create` on `pods` (or any workload kind) | Mount any SA in the namespace via `serviceAccountName` → steal its token; hostPath if PSA absent |
| `pods/exec`, `pods/attach` | Shell into running pods → read mounted tokens/secrets live |
| `escalate` verb on roles | Grant a role more permissions than the grantor holds |
| `bind` verb on roles/clusterroles | Bind existing powerful roles (incl. cluster-admin) to own SA |
| `impersonate` on users/groups/serviceaccounts | Act as any identity, incl. system:masters |
| wildcard `verbs: ["*"]` / `resources: ["*"]` / `apiGroups: ["*"]` | Unbounded; also silently gains new resource types added later |

Rules: Role/RoleBinding over ClusterRole/ClusterRoleBinding; explicit verbs and resources; `resourceNames` where possible; empty `apiGroups: [""]` = core group only, not "all". Dedicated SA per workload; `automountServiceAccountToken: false` on both the SA and pod spec; projected token volume (`audience` + `expirationSeconds`) when API access is needed.

Secrets: base64 ≠ encryption — plaintext in etcd unless `EncryptionConfiguration` (aescbc/secretbox, NOT `identity`) is set. Prefer external-secrets-operator or sealed-secrets. Mount as files, not env vars (env visible in `kubectl describe`, process env, crash dumps; file mounts rotate without restart).

Audit policy floor: `Metadata` level for `secrets`/`configmaps`; `RequestResponse` for `pods/exec` and `pods/attach`.

## Network exposure taxonomy

Default cluster networking is flat: every pod reaches every pod on every port. No NetworkPolicy = allow-all (policies are additive allows).

| Path | Exposure | Verdict |
|---|---|---|
| `ClusterIP` | In-cluster only | Default; set explicitly to document intent |
| `NodePort` | Every node IP, fixed port | Avoid in prod; block in shared clusters via quota `services.nodeports: "0"` |
| `LoadBalancer` | Public/cloud LB per Service | Only with justification; prefer ClusterIP + Ingress |
| `hostNetwork: true` | Node network namespace; **bypasses NetworkPolicy entirely** | Infra components only |
| Ingress w/o TLS | Plaintext at edge | Require `tls` block + `ingressClassName` |

Default-deny + DNS egress pattern (both required — deny without DNS allow breaks all resolution):

```yaml
kind: NetworkPolicy
metadata: {name: default-deny-all}
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]   # egress too — ingress-only still allows all outbound
---
kind: NetworkPolicy
metadata: {name: allow-dns}
spec:
  podSelector: {}
  policyTypes: [Egress]
  egress:
    - to: [{namespaceSelector: {matchLabels: {kubernetes.io/metadata.name: kube-system}}}]
      ports: [{protocol: UDP, port: 53}, {protocol: TCP, port: 53}]
```

- ❌ `namespaceSelector` and `podSelector` as separate `from` items when AND intended — separate list items = OR; same item = AND
- ❌ External egress via bare `cidr: 0.0.0.0/0` — add `except: [10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16]`
- ✅ Egress gateways / DNS policies to limit exfiltration paths beyond L3/4

## What namespaces do NOT isolate

- **Node/kernel**: pods from all namespaces share node kernel, CPU, memory, disk — container escape or noisy neighbor crosses tenants. Hard isolation needs node pools/taints or sandboxed runtimes (gVisor, Kata).
- **Cluster-scoped resources**: ClusterRoles, CRDs, PVs, Nodes visible cluster-wide.
- **Network**: open until a NetworkPolicy exists.
- **Quota**: unbounded until ResourceQuota exists; quota then rejects pods without requests/limits — pair with LimitRange defaults.

Tenant namespace floor: PSA labels + ResourceQuota (incl. `services.loadbalancers`, `services.nodeports: "0"`) + LimitRange + default-deny NetworkPolicy + namespace-scoped RBAC + per-namespace SAs. Separate clusters when: different compliance domains (PCI vs non), external customers, zero cross-tenant blast-radius tolerance, different k8s versions.

## Read-only audit one-liners

```bash
# Workloads not enforcing non-root / privileged / hostPath / hostNetwork
kubectl get pods -A -o json | jq '.items[] | select(.spec.containers[].securityContext.runAsNonRoot != true) | .metadata.name'
kubectl get pods -A -o json | jq '.items[] | select(.spec.containers[].securityContext.privileged == true) | "\(.metadata.namespace)/\(.metadata.name)"'
kubectl get pods -A -o json | jq '.items[] | select(.spec.volumes[]?.hostPath != null) | "\(.metadata.namespace)/\(.metadata.name)"'
kubectl get pods -A -o json | jq -r '.items[] | select(.spec.hostNetwork == true) | "\(.metadata.namespace)/\(.metadata.name)"'

# PSA labels on a namespace
kubectl get namespace <ns> -o jsonpath='{.metadata.labels}' | jq .

# RBAC: cluster-admin bindings, wildcards, default-SA pods, effective permissions
kubectl get clusterrolebindings -o json | jq -r '.items[] | select(.roleRef.name=="cluster-admin") | .metadata.name + " -> " + (.subjects[]? | .kind + "/" + .name)'
kubectl get roles,clusterroles -A -o json | jq -r '.items[] | select(.rules[]? | .verbs[]? == "*" or .resources[]? == "*") | .metadata.namespace + "/" + .metadata.name'
kubectl get pods -A -o json | jq -r '.items[] | select(.spec.serviceAccountName == "default" or .spec.serviceAccountName == null) | .metadata.namespace + "/" + .metadata.name'
kubectl auth can-i --list --as=system:serviceaccount:<ns>:<sa> -n <ns>
kubectl auth can-i get secrets --as=system:serviceaccount:<ns>:<sa> -n <ns>

# Secrets exposed as env vars
kubectl get pods -A -o json | jq -r '.items[] | .metadata.namespace + "/" + .metadata.name as $pod | .spec.containers[]?.env[]? | select(.valueFrom.secretKeyRef != null) | $pod + " env:" + .name'

# Network exposure: NodePort/LoadBalancer services; NetworkPolicy presence; selector→endpoints sanity
kubectl get svc -A -o json | jq -r '.items[] | select(.spec.type == "NodePort" or .spec.type == "LoadBalancer") | "\(.metadata.namespace)/\(.metadata.name): \(.spec.type)"'
kubectl get networkpolicy -n <ns>
kubectl get endpoints <svc> -n <ns>            # <none> = selector matches no pods
kubectl get secret <tls-secret> -n <ns> -o jsonpath='{.type}'   # expect kubernetes.io/tls
```
