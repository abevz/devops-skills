# Cilium Gateway API, Ingress, Egress Gateway, LB-IPAM, BGP, L2

Self-authored reference (no third-party source). Cilium's north-south and egress features —
often the reason to drop a separate ingress controller / MetalLB. Deployment-planning: propose
config, don't apply.

## Gateway API and Ingress

Cilium implements **Gateway API** (and can serve Ingress) natively — no separate ingress
controller pod:

```yaml
gatewayAPI: {enabled: true}          # needs Gateway API CRDs installed first
# or legacy Ingress:
ingressController: {enabled: true, loadbalancerMode: shared}   # shared vs dedicated LB per Ingress
```

- Gateway API is the forward-looking choice (see the broader Gateway API migration story);
  Cilium supports GAMMA (mesh) routes too. CRDs must be installed before enabling.
- `loadbalancerMode`: `shared` (one LB IP for all ingress, cheaper) vs `dedicated` (per-resource
  LB). Shared is the default answer; dedicated for isolation/cost-attribution.
- Needs kube-proxy replacement and a way to get an external IP for the Gateway/Ingress Service —
  cloud LB, or Cilium LB-IPAM + BGP/L2 below on bare metal.

## LB-IPAM — LoadBalancer IPs without a cloud

On bare metal, `type: LoadBalancer` Services have no cloud LB to assign an IP. Cilium LB-IPAM
does it (replaces MetalLB):

```yaml
apiVersion: cilium.io/v2alpha1
kind: CiliumLoadBalancerIPPool
spec:
  blocks: [{cidr: "192.0.2.0/24"}]
  serviceSelector: {matchLabels: {...}}    # optional: which services draw from this pool
```

Assigning an IP is only half — the network must **route/announce** it (BGP or L2 below), or the
IP is assigned but unreachable.

## BGP — advertise service/pod IPs to the network

Cilium BGP Control Plane peers with your routers to advertise LoadBalancer IPs (and optionally
pod CIDRs):

```yaml
# newer BGP Control Plane (CiliumBGPClusterConfig / PeerConfig / Advertisement CRDs)
apiVersion: cilium.io/v2alpha1
kind: CiliumBGPClusterConfig
spec:
  bgpInstances:
    - name: instance-65000
      localASN: 65000
      peers:
        - name: tor-switch
          peerASN: 65001
          peerAddress: 192.0.2.1
```

- Two generations exist: legacy `CiliumBGPPeeringPolicy` vs the newer multi-CRD BGP Control
  Plane (`CiliumBGPClusterConfig`/`PeerConfig`/`Advertisement`). Check the cluster's Cilium
  version — the CRDs and capabilities differ; don't mix.
- Advertise LB service IPs (and pod CIDR for native routing without overlay). Verify with
  `cilium bgp peers` / `cilium bgp routes` (read-only).
- Requires coordination with network/router owners (ASNs, peering) — a cross-team change, flag
  that in the plan.

## L2 announcements — ARP for LB IPs on a flat network

When there's no BGP router, `CiliumL2AnnouncementPolicy` makes a node answer ARP for the LB IP
(like MetalLB L2 mode):

```yaml
apiVersion: cilium.io/v2alpha1
kind: CiliumL2AnnouncementPolicy
spec:
  interfaces: ["eth0"]
  externalIPs: true
  loadBalancerIPs: true
```

- Needs kube-proxy replacement; one node "owns" each IP (failover on node loss, brief blip).
- Simpler than BGP, but L2-only (single broadcast domain) and no true load-balancing across
  nodes — fine for homelab/small bare-metal, not for scale.

## Egress Gateway — static source IP for pod egress

Route selected pods' egress through a specific gateway node with a **fixed SNAT IP** — for
legacy firewalls/partners that allowlist a source IP:

```yaml
apiVersion: cilium.io/v2
kind: CiliumEgressGatewayPolicy
spec:
  selectors:
    - podSelector: {matchLabels: {app: partner-connector}}
  destinationCIDRs: ["203.0.113.0/24"]
  egressGateway:
    nodeSelector: {matchLabels: {egress-node: "true"}}
    egressIP: 192.0.2.50
```

- Requires kube-proxy replacement. The gateway node is a chokepoint/SPOF for that egress —
  plan HA (multiple egress nodes) if the path is critical.
- Use case: "our pods must appear as one stable IP to an external system." Don't reach for it
  otherwise — it adds a hop and a failure domain.

## Choosing the north-south stack (bare metal)

| Need | Use |
|---|---|
| LB IPs assigned | LB-IPAM (CiliumLoadBalancerIPPool) |
| Announce IPs, have BGP routers | BGP Control Plane |
| Announce IPs, flat L2, no routers | L2 announcements |
| HTTP routing / ingress | Gateway API (preferred) or Ingress controller |
| Fixed egress source IP | Egress Gateway (+ HA egress nodes) |

On cloud, the cloud LB usually covers LB-IPAM/announcement — these shine on bare metal/homelab.
