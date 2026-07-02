# Istio security: mTLS, authorization, identity

Self-authored reference (no third-source). mTLS, AuthorizationPolicy, JWT, identity/CA. (mTLS
handshake failures → `istio-debug`.) Propose config; never disable mTLS mesh-wide as a shortcut.

## mTLS via PeerAuthentication

```yaml
kind: PeerAuthentication
metadata: {name: default, namespace: payments}     # namespace-scope; omit ns + istio-system name = mesh-wide
spec:
  mtls: {mode: STRICT}                              # STRICT | PERMISSIVE | DISABLE
  # portLevelMtls: {8080: {mode: PERMISSIVE}}       # per-port override
```

| Mode | Accepts | Use |
|---|---|---|
| PERMISSIVE | mTLS **and** plaintext | Migration/transition — the default while onboarding |
| STRICT | mTLS only | Target state — plaintext (non-mesh) clients rejected |
| DISABLE | plaintext only | Special cases (a port that must stay plaintext) |

- Move **PERMISSIVE → observe → STRICT**, per namespace, never a blind mesh-wide flip: STRICT
  with a legacy plaintext client = instant connection resets (`istio-debug` mTLS matrix).
- Scope precedence: workload > namespace > mesh (most specific wins). A mesh-wide STRICT with a
  namespace PERMISSIVE override is a valid migration pattern.
- mTLS is **mesh-internal transport**; edge/client TLS terminates at the ingress gateway — don't
  conflate the two.

## AuthorizationPolicy

```yaml
kind: AuthorizationPolicy
metadata: {name: allow-frontend, namespace: payments}
spec:
  selector: {matchLabels: {app: api}}
  action: ALLOW                                      # ALLOW | DENY | CUSTOM | AUDIT
  rules:
    - from: [{source: {principals: ["cluster.local/ns/payments/sa/frontend"]}}]
      to: [{operation: {methods: [GET, POST], paths: ["/v1/*"]}}]
```

- **Default-deny pattern**: an empty `ALLOW` policy (`action: ALLOW`, no rules) on a namespace/
  workload denies everything not explicitly allowed — the mesh-authz equivalent of a default-deny
  NetworkPolicy. Apply it, then add explicit ALLOWs.
- Evaluation: `CUSTOM` (ext authz) → `DENY` → `ALLOW`. DENY wins over ALLOW; if any ALLOW policy
  matches a workload, everything else is denied by default.
- `principals` are SPIFFE identities (the workload's SA), not IPs — identity-based authz. L7
  conditions (paths/methods/headers) need a sidecar or an ambient **waypoint** (L4-only ambient
  can't enforce path rules — see sidecar-vs-ambient.md).
- `AUDIT` action logs would-be-denies without enforcing — use it to validate a policy before
  switching it to DENY/ALLOW (audit-before-enforce, like admission policy).

## JWT via RequestAuthentication (+ AuthorizationPolicy)

```yaml
kind: RequestAuthentication
spec:
  selector: {matchLabels: {app: api}}
  jwtRules: [{issuer: "https://idp.example.com", jwksUri: "https://idp.example.com/jwks"}]
```

- `RequestAuthentication` **validates** JWTs if present but doesn't require them — it only rejects
  *invalid* tokens. To **require** a valid token, pair with an AuthorizationPolicy requiring
  `requestPrincipals: ["*"]`. This two-step is the common miss: RequestAuthentication alone lets
  no-token requests through.

## Identity and certificates

- Workload identity is **SPIFFE**: `spiffe://<trust-domain>/ns/<ns>/sa/<serviceaccount>` — derived
  from the pod's ServiceAccount; this is what mTLS and `principals` use. Least-privilege SAs per
  workload make authz meaningful.
- **istiod is the CA** by default (issues/rotates workload certs automatically, short-lived —
  ~24h, auto-renewed). No manual cert management for workloads.
- Custom/enterprise CA: plug in intermediate CA certs (the `cacerts` Secret) so istiod issues
  from your org PKI, or integrate an external CA / cert-manager (`istio-csr`) — for a shared trust
  domain across clusters or compliance. The `cacerts` key material is crown-jewels — reference,
  never inline.
- Trust domain must be consistent across clusters you want to mTLS-connect (multi-cluster/mesh
  federation).

## Verify (read-only)

```bash
istioctl analyze -A
istioctl x describe pod <pod>            # effective mTLS mode + authz policies per port
istioctl proxy-config secret <pod>      # cert presence/expiry (not the key material)
```
