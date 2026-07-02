# Cilium network policies (L3/L4/L7)

Self-authored reference (no third-party source). CiliumNetworkPolicy beyond native
NetworkPolicy. Propose policies; validate with Hubble before enforcing — a wrong selector cuts
prod traffic instantly.

## Why CNP over native NetworkPolicy

| Capability | native NetworkPolicy | CiliumNetworkPolicy |
|---|---|---|
| Identity-based (labels, not IP) | via selectors, IP under the hood | ✅ true identity (survives IP churn) |
| L7 (HTTP/DNS/Kafka) | ❌ | ✅ |
| Cluster-wide | ❌ (namespaced only) | ✅ `CiliumClusterwideNetworkPolicy` |
| Entities (world, cluster, host, kube-apiserver) | ❌ | ✅ |
| DNS-aware egress (`toFQDNs`) | ❌ | ✅ |
| Explicit deny | ❌ (deny by absence only) | ✅ deny rules |
| Host firewall (node itself) | ❌ | ✅ |

They **combine**: native NetworkPolicy and CNP both apply; a default-deny in either flips the
namespace to allowlist mode (a frequent "why is it blocked" cause — check both, see
`cilium-debug`).

## L3/L4 structure

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata: {name: api-allow, namespace: payments}
spec:
  endpointSelector: {matchLabels: {app: api}}     # who this policy governs
  ingress:
    - fromEndpoints: [{matchLabels: {app: frontend}}]
      toPorts: [{ports: [{port: "8080", protocol: TCP}]}]
  egress:
    - toEndpoints: [{matchLabels: {app: db}}]
      toPorts: [{ports: [{port: "5432", protocol: TCP}]}]
```

Direction matters: egress on the source AND ingress on the destination must both allow a flow.
`endpointSelector: {}` selects every pod in the namespace → the default-deny building block.

## Entities — the special sources/destinations

`fromEntities`/`toEntities`: `world` (outside cluster), `cluster` (anything in-cluster), `host`
(the local node), `remote-node`, `kube-apiserver`, `health`, `all`. Common needs:
- Allow DNS to kube-dns, allow egress to `kube-apiserver` for controllers, allow `world` on 443
  for external APIs (better: `toFQDNs`).
- ❌ `toEntities: [world]` wide open — scope to ports, or use FQDN rules.

## L7 policies (the Cilium differentiator)

L7 rules route matched traffic through an Envoy proxy in the datapath:

```yaml
  egress:
    - toEndpoints: [{matchLabels: {app: api}}]
      toPorts:
        - ports: [{port: "8080", protocol: TCP}]
          rules:
            http:
              - method: "GET"
                path: "/v1/.*"
              - method: "POST"
                path: "/v1/orders"
```

- HTTP (method/path/headers/host), Kafka (topic/role), and DNS are the L7 protocols.
- L7 cost: traffic goes through the proxy → latency + the proxy is now in the path. Use L7 where
  it earns its keep (API authz boundaries), not everywhere.
- A partial L7 rule denies non-matching requests on that port with an L7 verdict (Hubble shows
  `DROPPED (L7)`), while still allowing the L4 connection — subtle in debugging.

## DNS-aware egress (`toFQDNs`)

```yaml
  egress:
    - toEndpoints: [{matchLabels: {"k8s:io.kubernetes.pod.namespace": kube-system, k8s-app: kube-dns}}]
      toPorts:
        - ports: [{port: "53", protocol: ANY}]
          rules: {dns: [{matchPattern: "*"}]}      # MUST allow DNS first
    - toFQDNs: [{matchName: "api.stripe.com"}, {matchPattern: "*.amazonaws.com"}]
      toPorts: [{ports: [{port: "443", protocol: TCP}]}]
```

- The **DNS visibility rule is mandatory and comes first** — Cilium learns FQDN→IP mappings only
  from DNS it proxies; without the `rules.dns` allow, `toFQDNs` never matches (top FQDN-policy
  bug). `matchPattern` matches one label level: `*.example.com` ≠ `a.b.example.com`.

## Default-deny and deny policies

- Default-deny per namespace: a CNP with `endpointSelector: {}` and empty `ingress`/`egress`
  (or use a native NetworkPolicy default-deny). Remember **egress** default-deny needs an
  explicit DNS allow or everything breaks.
- Explicit **deny** rules (`ingressDeny`/`egressDeny`) override allows — for carve-outs
  ("allow the namespace, but never to the metadata endpoint 169.254.169.254"). Deny wins over
  allow; use sparingly and document.

## Host firewall & clusterwide

- `CiliumClusterwideNetworkPolicy` + `nodeSelector` with host firewall (`hostFirewall.enabled`)
  protects the **nodes themselves** (SSH, kubelet ports) — powerful and easy to lock yourself out
  of a node with; stage carefully, keep console access.
- Clusterwide policies apply across all namespaces — great for baseline (allow DNS everywhere,
  deny egress to metadata IP everywhere), dangerous as a blunt instrument.

## Validate before enforce

1. Author the policy but observe first: `hubble observe --verdict DROPPED --from-pod ns/pod`
   with the policy in place shows what it *would* break.
2. Cilium has a policy audit mode (`--policy-audit-mode` on the agent, or per-endpoint) that
   logs would-be drops without enforcing — the equivalent of admission audit mode.
3. Roll namespace by namespace, watching Hubble drops, before cluster-wide default-deny.
   ❌ apply a cluster-wide default-deny egress and discover the DNS gap in production.
