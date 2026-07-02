# Vault: Kubernetes consumption and disaster recovery

Self-authored reference (no third-party source). Getting Vault secrets into pods, and not losing
Vault. Propose config; never read values or run restores live.

## Getting secrets into Kubernetes — four patterns

| Pattern | How | Secret becomes a k8s Secret? | Fit |
|---|---|---|---|
| External Secrets Operator (ESO) | ESO's Vault `SecretStore` syncs Vault → k8s `Secret` | Yes | Default when you want a normal Secret and standard consumption (see `secrets-management`) |
| Vault Agent Injector | Sidecar/init injects secrets as **files** into the pod via annotations | No (files in pod) | App reads files; secret never a k8s Secret object |
| Secrets Store CSI + Vault provider | CSI volume mounts secrets as files | Optionally mirrored to a Secret | CSI-standard shops |
| Vault Agent (app-managed) | App/agent authenticates and fetches directly | No | Apps that speak Vault natively, dynamic-lease renewal |

Decision:
- Want a plain `Secret` other tooling consumes → **ESO** (keeps Vault out of the app path).
- Want secrets to **never become a k8s Secret** (only files in the pod) → **Agent Injector** or
  **CSI** — smaller exposure (no Secret object to RBAC-leak), at the cost of app-side file
  handling.
- Need **dynamic lease renewal** (DB creds expiring hourly) → Vault Agent handles renew/re-fetch;
  ESO re-syncs on interval but the pod still must pick up the change (see the rotation trap in
  `secrets-management`).

All four use **Kubernetes auth** (pod SA → Vault role → policy); none should carry a static
Vault token in a Secret.

## Agent Injector shape

```yaml
# pod annotations (shape)
vault.hashicorp.com/agent-inject: "true"
vault.hashicorp.com/role: payments
vault.hashicorp.com/agent-inject-secret-db: database/creds/payments-ro
# renders to /vault/secrets/db in the pod; agent renews the lease
```

The injector mutating webhook must be healthy for annotated pods to start — an unhealthy injector
+ `fail`-mode webhook blocks those pods (webhook-availability tradeoff, cf. `admission-policy-review`).

## Disaster recovery — the part that ends careers if skipped

Vault storage (Raft) holds the encrypted root of trust and all secrets/leases. DR = snapshots +
a tested restore + the unseal path.

```bash
# Raft snapshot (integrated storage) — schedule it, store OFF the cluster, encrypted
vault operator raft snapshot save vault-$(date +%F).snap        # read side safe
vault operator raft snapshot restore vault-2026-07-02.snap      # DESTRUCTIVE — confirm first
```

- Snapshot on a schedule (CronJob / external), retained off-box; a snapshot on the same PVC that
  died is no snapshot.
- **Restore needs the same seal**: a snapshot restored to a Vault with a different auto-unseal
  KMS key can't be unsealed. The KMS key (or recovery keys) is part of the backup story — losing
  the KMS key + storage = unrecoverable. Document KMS key custody/replication alongside snapshots.
- **Test the restore** into a scratch Vault and confirm it unseals and secrets read — an untested
  Vault backup is the highest-stakes untested backup you own (cf. `production-readiness`).
- Enterprise DR replication (a warm standby cluster) is the low-RTO option; snapshots are the
  baseline everyone needs regardless.

## Monitoring

- Telemetry → Prometheus (`vault_core_unsealed` = is it sealed!, `vault_expire_num_leases`,
  `vault_token_count`, request rates/latency, `vault_core_leadership_lost`).
- Alert on: **sealed** (unsealed==0), leadership flaps, lease-count explosion, and the audit
  device failing (below).
- Audit device: enable an audit backend (file/socket) — Vault **refuses requests if all audit
  devices fail**, so a full disk on the audit log = outage; monitor it.

## Verify (read-only)

```bash
vault status                              # sealed / HA
vault operator raft snapshot save -      # can we even snapshot? (writes to stdout/file)
vault audit list                          # audit devices enabled?
kubectl get mutatingwebhookconfiguration | grep vault    # injector health
```
