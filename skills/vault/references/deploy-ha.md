# Vault deployment, HA, and seal

Self-authored reference (no third-party source). Storage, HA, seal/auto-unseal, upgrades.
Deployment-planning: propose config, never unseal or apply to a live Vault.

## Storage backend

| Backend | HA | Note |
|---|---|---|
| Integrated Storage (Raft) | ✅ built-in | **Default choice** — no external dependency, snapshots built in, data on each node |
| Consul | ✅ | Legacy pattern; adds a Consul cluster to operate — only if you already run Consul |
| Cloud (S3/GCS/etc.) | ❌ (no HA on their own) | Avoid for HA; Raft superseded these |

Raft is the modern answer: `storage "raft"` with a `retry_join` per peer, odd node count (3 or 5)
for quorum. On Kubernetes, the Helm chart's `server.ha.raft.enabled: true` runs a StatefulSet
with a PVC per pod.

## HA topology

- 3 nodes (tolerate 1 loss) or 5 (tolerate 2) — odd for quorum. One **active**, others
  **standby** forwarding to active (or performance standbys serving reads with Enterprise).
- Anti-affinity across nodes/zones; a Vault where all replicas share a node/zone isn't HA.
- Load balancer / Service in front routes to the active node (standbys redirect); the chart wires
  this. Health checks must use `/v1/sys/health` with the right params or standbys look unhealthy.

## Seal and auto-unseal — the availability crux

Vault starts **sealed**; it can't serve until unsealed (decrypt the storage key).

| Seal | Unseal | Use |
|---|---|---|
| Shamir (default) | Manual: k-of-n key holders each enter a share on every start/restart | ❌ operationally painful — a restart pages humans |
| Auto-unseal (cloud KMS / HSM / Transit) | Vault decrypts the root key via the KMS automatically | ✅ operational default |

```hcl
# auto-unseal via AWS KMS (example shape)
seal "awskms" {
  region     = "eu-central-1"
  kms_key_id = "arn:aws:kms:...:key/..."     # reference; access via instance/IRSA role
}
```

- Auto-unseal means a pod restart recovers **without human intervention** — essential for a
  StatefulSet that reschedules. Manual unseal on Kubernetes is a trap (every pod restart seals).
- With auto-unseal, the Shamir keys become **recovery keys** (for critical ops like root
  regeneration) — still generated, still split, still custody-critical, just not needed on every
  start.
- The KMS/HSM is now a hard dependency: if it's unreachable, Vault can't unseal. Factor its
  availability/region into the DR plan.

## Key custody (non-negotiable process, not a technical toggle)

- Unseal/recovery keys are split (Shamir, e.g. 5 shares, threshold 3) across **different people/
  systems** — never all in one place, never in the repo, never in one person's password manager.
- Root token: used once for initial config, then **revoked** (`vault token revoke`); regenerate
  via recovery keys for break-glass. A long-lived root token is the top audit finding.

## OpenBao note

OpenBao is the Linux-Foundation fork of Vault (MPL, post-BUSL). Largely config-compatible for
core (Raft, KV, k8s auth, PKI, transit); verify specific engine/feature parity per version if
choosing it for licensing reasons. The patterns in these references apply to both — note which
you're on in the baseline.

## Upgrades

- Raft: rolling upgrade — upgrade standbys first, step-down the active last
  (`vault operator step-down`), one version-line at a time; take a snapshot first.
- Read release notes for storage/seal changes; pin the chart/image version.
- Test in a non-prod Vault; a botched active-node upgrade can seal the cluster.

## Verify (read-only)

```bash
vault status                       # sealed? HA mode? active/standby, storage type
vault operator raft list-peers     # quorum health
vault operator members             # node roles (Enterprise)
```
