# Cilium transparent encryption

Self-authored reference (no third-party source). Node-to-node pod-traffic encryption: WireGuard
vs IPsec, what it protects, and its cost. Deployment-planning: propose values, don't apply.

## WireGuard vs IPsec

| | WireGuard | IPsec |
|---|---|---|
| Config complexity | Low — Cilium manages keys, one toggle | Higher — pre-shared key in a k8s Secret, rotation to manage |
| Kernel | WireGuard module (mainline 5.6+, or module) | XFRM (widely present) |
| Port | `51871/UDP` | ESP / IKE |
| FIPS/compliance | Not FIPS-validated | Available for FIPS requirements |
| Perf | Generally faster/simpler | More overhead, more tunables |
| Default recommendation | ✅ preferred for most | Only when FIPS/IPsec mandated |

```yaml
encryption:
  enabled: true
  type: wireguard            # or ipsec
  nodeEncryption: false      # also encrypt node-to-node (host) traffic, not just pod-to-pod
# ipsec extra: encryption.ipsec.keyFile / a cilium-ipsec-keys Secret + rotation
```

## What it actually encrypts (and doesn't)

- Encrypts **pod-to-pod traffic across nodes** (and node/host traffic with `nodeEncryption`).
  Same-node pod traffic never leaves the box.
- It is **transport encryption between nodes**, not application mTLS. It does NOT authenticate
  workloads to each other or replace a service mesh's identity/mTLS — say this plainly when
  someone treats "encryption: on" as "zero trust done."
- With native routing, confirm the underlay doesn't strip/forbid the encryption protocol
  (WireGuard UDP port, ESP for IPsec) — cloud/node firewalls blocking it = encrypted traffic
  blackholes while unencrypted worked.

## Cost and gotchas

- MTU: encryption headers shrink usable MTU further (on top of any tunnel overhead) — Cilium
  adjusts MTU, but mismatches with the underlay surface as large-payload hangs.
- CPU: real per-node overhead at high throughput; size node CPU accordingly for encryption-heavy
  clusters.
- Enablement is cluster-wide and touches the datapath on every node → stage it (a subset of
  nodes / a canary) with connectivity checks; during rollout mixed encrypted/unencrypted nodes
  can drop traffic until all agents converge.
- IPsec key rotation is an operational task (rotate the Secret, agents pick up new SPI) — plan
  it; a stale/expired key is a cluster-wide outage.

## Verify (read-only)

```bash
cilium status | grep -i encryption            # mode + node count
cilium status --verbose | grep -iA3 Encryption
# WireGuard peer/keys visible via the agent; IPsec via `cilium status` XFRM counters
```

Recommend encryption when there's a real requirement (untrusted underlay, compliance, multi-
tenant traffic isolation) — not by default; it's cost without benefit on a trusted private
underlay where the threat model doesn't include node-to-node sniffing.
