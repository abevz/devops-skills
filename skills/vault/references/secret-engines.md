# Vault secret engines

Self-authored reference (no third-party source). Choosing and operating engines. Prefer dynamic/
short-lived over static. Propose config; never read secret values.

## Engine choice

| Engine | Gives | Prefer when |
|---|---|---|
| KV v2 | Versioned static secrets | Config secrets with no dynamic source (API keys you're given) |
| Database | **Dynamic** DB creds, auto-expiring | Any DB — issue per-app short-lived users instead of a shared password |
| AWS / GCP / Azure | **Dynamic** cloud creds | Short-lived cloud access instead of static IAM keys |
| PKI | On-demand certs | Internal TLS/mTLS, service certs |
| Transit | Encryption-as-a-service (Vault holds the key, encrypts/decrypts for you) | App needs to encrypt data without handling keys |
| SSH | Signed SSH certs / OTP | Just-in-time SSH access |

Guiding rule: **dynamic beats static.** A static DB password in KV is a rotation liability; a
`database/creds/<role>` lease is a username/password that Vault created and will revoke on TTL.

## KV v2 essentials

- Versioned: `kv put`/`kv get` with version history, `kv rollback`, soft-delete + `destroy`.
- Remember the `data/` vs `metadata/` path split for policies (see auth-policies.md).
- `max_versions`, `cas_required` (compare-and-set to prevent blind overwrite), `delete_version_after`.
- ❌ treating KV as the default for everything — reach for a dynamic engine first if the backend
  supports it.

## Dynamic database creds

```hcl
# config (shape): Vault gets an admin cred to the DB and mints short-lived users
# vault write database/roles/payments-ro \
#   db_name=appdb creation_statements="CREATE ROLE ..." \
#   default_ttl=1h max_ttl=24h
```

- The app reads `database/creds/payments-ro` → gets a unique user/pass with a **lease**; Vault
  drops the DB user when the lease expires/revokes.
- The app (or its Vault Agent) must **renew or re-fetch** before lease expiry, or the DB user
  vanishes mid-run — the dynamic-secret equivalent of the rotation-reaches-the-pod problem.
- Vault needs a privileged DB credential to create/drop users — scope it, and it's itself a
  high-value secret.

## PKI

- Root CA (offline/short-lived) → intermediate CA in Vault → roles issue leaf certs with bounded
  TTL and allowed domains. ❌ long-TTL leaf certs — the point of Vault PKI is short-lived,
  auto-renewed certs.
- Roles constrain `allowed_domains`, `max_ttl`, key type; apps/agents request certs on demand.
- CRL/OCSP and rotation of the intermediate are operational tasks — plan them.

## Transit (encryption as a service)

- App sends plaintext → Vault returns ciphertext (and back); the key never leaves Vault. Supports
  key rotation (new version, old still decrypts) and re-wrapping.
- Good for app-layer encryption without the app managing keys; the app needs `transit/encrypt/<key>`
  / `decrypt` capability, nothing more.

## Leases, TTL, rotation

- Everything dynamic has a **lease** (TTL + max-TTL). Monitor lease counts
  (`vault_expire_num_leases`) — a lease explosion (apps fetching without renewing) is a real
  outage/perf source.
- Static secret rotation: KV rotation is manual/external; dynamic engines rotate by design.
- Root/admin credentials the engines use (DB admin, cloud IAM) should themselves be rotated —
  Vault can rotate its own DB root (`database/rotate-root`) so even the operator can't know it.

## Verify (read-only)

```bash
vault secrets list                       # mounted engines
vault read database/roles/payments-ro    # role config (not a live credential)
vault list sys/leases/lookup/database/creds/payments-ro   # outstanding leases (no values)
```
