# Hubble observability

Self-authored reference (no third-party source). Deploying and using Hubble for flow visibility
and metrics. Deployment-planning: propose Helm values, don't apply.

## Components

| Component | Role | Enable |
|---|---|---|
| Hubble (in-agent) | Per-node flow data from the eBPF datapath | `hubble.enabled: true` |
| Hubble Relay | Cluster-wide aggregation across all agents | `hubble.relay.enabled: true` |
| Hubble UI | Web service map + flow browser | `hubble.ui.enabled: true` |
| `hubble` CLI | Query flows from a terminal | client-side |

Without Relay, `hubble observe` only sees the local node's flows. Relay is what makes
cluster-wide queries and the UI work.

## Flow observability

```bash
hubble observe --namespace payments --follow
hubble observe --from-pod payments/api --to-pod payments/db --verdict DROPPED
hubble observe --protocol dns --type l7            # L7 needs L7 policy or visibility
hubble observe --verdict DROPPED --last 500        # what's being blocked and why
```

- The **drop reason** field is the single most useful signal for policy debugging (see
  `cilium-debug`): `Policy denied`, `Policy denied (L7)`, DNS, CT.
- L7 flow visibility (HTTP paths, DNS queries) appears only when an L7 policy proxies that
  traffic, or via visibility annotations (`io.cilium.proxy-visibility`) — L3/L4-only traffic
  shows connections, not HTTP.

## Metrics → Prometheus

```yaml
hubble:
  metrics:
    enabled: [dns, drop, tcp, flow, port-distribution, icmp, "http:destinationContext=pod"]
    serviceMonitor: {enabled: true}     # if running the Prometheus Operator
```

- Enables an OpenMetrics endpoint on the agent; the `serviceMonitor` wires it into
  kube-prometheus-stack (mind the selector-label trap — see the prometheus-stack reference).
- Cardinality warning: `http`/`flow` metrics with high-context labels (`sourceContext`/
  `destinationContext=pod`) explode series in big clusters — start with `identity`/`namespace`
  context, add pod-level only where needed (ties to the cardinality section of
  `observability-review`).
- Dashboards: the community "Hubble" + "Cilium Metrics" Grafana dashboards; provision as code
  (see `grafana-dashboards`).

## Flow export (audit / long-term)

`hubble.export` writes flow logs (JSON) for retention/SIEM beyond the in-memory ring buffer
(which is small and rolls fast). Configure field masks/filters to avoid writing every flow —
export the verdicts and namespaces you'd actually investigate, not the firehose.

## What Hubble is and isn't

- ✅ network flow observability (who talked to whom, allowed/denied, L7 verdicts) — fills the
  "was it the network?" gap in incident response.
- ❌ not a metrics/tracing replacement — it complements Prometheus (rates/latency) and tracing
  (request paths); correlate Hubble drops with app metrics by pod/namespace and deploy
  annotations.
- Security angle: Hubble flow data is how you *verify* a network policy does what you intended,
  and how runtime investigations confirm "did this compromised pod talk to anything it
  shouldn't" (feeds `runtime-security-review`).
