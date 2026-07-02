# Cilium drop-reason playbooks

`hubble observe --verdict DROPPED -f` (scoped with `--from-pod`/`--to-pod`) names the drop
reason — route by it. All commands read-only.

## Drop reason → cause → fix

| Hubble drop reason | Cause | Diagnose / fix |
|---|---|---|
| `Policy denied` | No policy allows this flow | See "Policy denied walkthrough" below — direction and policy-kind matrix |
| `Policy denied (L7)` | L4 allowed, but an L7 (HTTP/Kafka/DNS) rule rejects the request | `hubble observe --type l7` shows the L7 verdict; check `rules.http` methods/paths against the actual request |
| `DNS proxy` denials / FQDN not resolving | `toFQDNs` policy without a DNS allow rule, or name never resolved through the proxy | The pod must be allowed UDP/TCP 53 to kube-dns with `rules.dns` (`matchPattern: "*"` at minimum) — the FQDN allowlist only learns IPs from DNS answers it can see |
| `Stale or unroutable identity` | Endpoint identity churn (labels changed, agent restart mid-flight) | `cilium identity get <id>`; usually transient — persistent means identity/label instability, check for rapidly-changing pod labels |
| `CT: Map insertion failed` / conntrack drops | Connection-tracking table full | `cilium status --verbose` → CT map pressure; raise `bpf-ct-global-*-max` or find the connection-leaking workload |
| `Missed tail call` / datapath internal | Agent/datapath bug or partial-upgrade state | Check agent versions across nodes (`kubectl get pods -n kube-system -l k8s-app=cilium -o jsonpath='{..image}'`); usually resolved by completing the rollout — agent restarts need user confirmation |
| `Unsupported L3 protocol` / `Invalid packet` | Malformed or unexpected traffic (often scanners/health checkers) | Usually ignorable noise; correlate source before chasing it |

## Policy denied walkthrough

The four questions, in order:

1. **Which direction dropped?** Hubble shows the drop at the enforcing endpoint. Egress drop at
   source = source's egress rules; ingress drop at destination = destination's ingress rules. A
   flow needs *both* allowed.
2. **What policies select each endpoint?** All three kinds combine:
   `kubectl get cnp,ccnp,netpol -A` — filter by the endpoints' namespaces. A default-deny in any
   of the three (including a plain NetworkPolicy someone forgot) flips the namespace to
   allowlist mode.
3. **Does the selector actually match?** `cilium endpoint list` on the pod's node → the
   endpoint's identity labels. Policies match *identity labels* — a selector on a label the pod
   doesn't carry (typo, `app` vs `app.kubernetes.io/name`) silently matches nothing.
4. **Is it the port/protocol, not the peer?** Policy can allow the peer but restrict `toPorts`.
   The hubble drop line shows the port — compare against the policy's port list.

Minimal-fix rule: add the narrowest allow (specific peer + port), never a broad
`{}`-selector allow, and verify with `hubble observe --verdict FORWARDED` after the user
applies it.

## Service / kube-proxy replacement issues

- **ClusterIP unreachable, pods fine**: `cilium service list` on the client's node — does the
  service have backends? Empty backends with Ready pods = endpoint propagation issue (agent out
  of sync; check `cilium status` sync sections).
- **NodePort/LoadBalancer**: with KPR enabled, node firewalls must allow the port on the node;
  `cilium service list` shows the frontend → backend mapping to confirm the datapath knows it.
- **hostPort not working**: only handled by Cilium when KPR/hostPort support is enabled — check
  `cilium status --verbose | grep HostPort`.

## Node-to-node / encryption issues

- `cilium-health status` (inside an agent pod) → node connectivity matrix; a single unreachable
  node points at underlay networking, not policy.
- Tunnel mode (VXLAN 8472/UDP, Geneve 6081/UDP) or WireGuard (51871/UDP) blocked by node/cloud
  firewall = cross-node pod traffic dead while same-node works — the signature is "works on one
  node, times out across nodes."
- Encryption mismatch after enabling WireGuard: nodes not yet rolled show unencrypted-vs-
  encrypted drops; complete the rollout before debugging individual flows.

## DNS-specific flows

`hubble observe --protocol dns` shows query/response pairs through the DNS proxy:

- Queries visible, responses `RCODE: NXDomain` → naming problem, not policy (see
  kubernetes-debug DNS playbook: FQDN vs short names, ndots).
- Queries absent entirely → egress to kube-dns blocked (step 1–4 above, port 53).
- Responses fine but connection to the resolved IP drops → the `toFQDNs` list doesn't cover the
  answered name (wildcards match labels, not dots: `*.example.com` ≠ `a.b.example.com`).
