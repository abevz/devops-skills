# Argo CD operations: backup/DR, notifications, monitoring, tuning

Self-authored reference (no third-party source). Running Argo CD day-2. Propose config; treat
import/restore as destructive (explicit confirmation).

## Backup and disaster recovery

Argo CD state = declarative config in git + a set of k8s Secrets/ConfigMaps that are NOT in git
(repo/cluster credentials, SSO secrets, the RBAC/settings ConfigMaps if managed in-cluster). DR
covers both:

| What | Where it lives | DR action |
|---|---|---|
| Applications / AppProjects / ApplicationSets | git (should be) | re-apply from git (app-of-apps) |
| Repo & cluster credentials | `argocd` namespace Secrets | back up / re-create from secrets manager |
| SSO client secret, admin, signing keys | `argocd-secret` | back up / re-provision |
| RBAC, settings, plugins | `argocd-cm`/`argocd-rbac-cm` | git if managed declaratively |

```bash
# Full logical export (contains SECRETS — encrypt at rest, restricted access)
argocd admin export -n argocd > argocd-backup.yaml       # read side is safe
argocd admin import -n argocd argocd-backup.yaml         # DESTRUCTIVE — overwrites; confirm first
```

- `argocd admin export` output includes credentials → treat the file as a secret (encrypt,
  don't drop in CI logs/artifacts).
- The clean DR target: everything declarative (apps in git, credentials in a secrets manager via
  ESO) so a rebuild is "install Argo CD → point at git → secrets sync" with no logical import
  needed. `admin export/import` is the fallback when some state is imperative.
- **Test the restore** — an untested Argo CD backup is the same trap as an untested DB backup
  (cross-ref `production-readiness`). Rebuild into a scratch cluster and confirm apps reconcile.

## Notifications

`argocd-notifications` (built-in controller) via triggers + templates in `argocd-notifications-cm`:

```yaml
# subscribe an app by annotation
metadata:
  annotations:
    notifications.argoproj.io/subscribe.on-sync-failed.slack: team-a-alerts
```

- Useful triggers: `on-sync-failed`, `on-health-degraded`, `on-sync-status-unknown`. Route
  degraded/failed to the team's channel; don't subscribe every app to `on-sync-succeeded` (noise).
- Webhook/Slack tokens come from `argocd-notifications-secret` — referenced, not inline.

## Monitoring

- Each component exposes Prometheus metrics; scrape via ServiceMonitors (mind the
  kube-prometheus-stack selector-label trap — see `observability-review/prometheus-stack.md`).
- Signals worth alerting on:
  - `argocd_app_info{sync_status="OutOfSync"}` / `health_status="Degraded"` sustained
  - `argocd_app_reconcile` duration climbing (controller/repo-server pressure)
  - repo-server OOM/restarts (render failures cascade to all apps)
  - controller sharding imbalance (one shard hot)
- The official Argo CD Grafana dashboards exist; provision as code (`grafana-dashboards`).
- Argo CD monitoring its own apps ≠ monitoring Argo CD — do both.

## Performance tuning

| Symptom | Lever |
|---|---|
| Slow reconcile across many apps | repo-server replicas + `--parallelismLimit`; `ApplyOutOfSyncOnly=true` |
| High API server load from Argo CD | increase reconcile timeout (`timeout.reconciliation`), reduce over-frequent polling; use webhooks for git events instead of polling |
| Big monorepo slow to render | shallow/filtered fetch, split sources, webhook-triggered refresh |
| Controller CPU high, many clusters | more shards (consistent-hashing) |
| Redis pressure | redis-ha; it's cache, but a hot cache stalls reconcile |

Prefer **git webhooks** over polling for responsiveness at scale (`argocd-server` webhook
endpoint + a secret) — polling every app every few minutes is the default and it's wasteful for
large installs.

## Multi-tenancy recap (ties to rbac-sso.md)

Real isolation = AppProject boundaries (repos/destinations/cluster-resource limits) + RBAC by
group + project-scoped tokens + optionally per-team Argo CD instances for hard isolation. A
single shared Argo CD with everything in `default` project is not multi-tenant, just multi-user.

## Verify (read-only)

```bash
argocd admin settings validate -n argocd
kubectl top pods -n argocd                       # repo-server/controller pressure
argocd app list -o wide                          # sync/health spread
kubectl get svc -n argocd argocd-metrics argocd-server-metrics argocd-repo-server -o name
```
