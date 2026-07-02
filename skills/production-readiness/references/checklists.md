# Production-readiness deep checklists per artifact type

Self-authored reference (no third-party source). Load the section matching what's being
reviewed; each item names the concrete failure it prevents.

## Kubernetes workload

Beyond the manifest-level review (see kubernetes-yaml-review skill — don't duplicate it here),
readiness adds the operational layer:

- **Rollback rehearsed**: `kubectl rollout undo` works only while old ReplicaSets exist
  (`revisionHistoryLimit` > 0) AND the old image tag still exists in the registry (retention
  policies silently delete it — the classic "rollback impossible" discovery mid-incident).
- **Config rollback couples with code rollback**: if config lives in a ConfigMap the deploy
  mutates in place, rolling back the Deployment doesn't roll back config — immutable ConfigMaps
  (name-hashed, e.g. Kustomize configMapGenerator) make rollback atomic.
- **Startup dependency ordering**: what happens when this service starts before its
  dependencies? (Init containers/retries vs crash-loop-until-ready — both work; *neither* being
  considered is the finding.)
- **Graceful shutdown verified under load**: preStop + terminationGracePeriodSeconds actually
  tested by deleting a pod mid-traffic and watching error rates, not assumed from YAML.
- **Capacity**: requests sized from load-test or prod-like data; HPA max tested (does the
  cluster have headroom for maxReplicas × requests, or does scale-up hit Pending at the worst
  moment?); PDB compatible with node drains (allowed disruptions ≥ 1 at steady state).

## Stateful service / database

- **Restore is the deliverable, not backup**: a restore has been executed to a scratch
  environment, timed (this number IS your RTO), and the restored data validated (app boots
  against it, row counts sane). Backup jobs green for months ≠ restorable.
- **Backup covers everything needed to rebuild**: data + schema migrations state + secrets/
  encryption keys (a backup encrypted with a key that only lived on the dead host is a brick).
- **PV reclaim policy**: `Retain` for anything you'd cry about; `Delete` reclaim + namespace
  deletion = data gone. StorageClass `allowVolumeExpansion: true` before you need it.
- **Failover story**: single-replica DB = accepted downtime (write it down as accepted risk);
  replicated = failover actually exercised, replication lag alerted on.
- **Migration discipline**: schema migrations forward-only-safe (old code runs against new
  schema during rollout) or the deploy strategy explicitly serializes them.

## Terraform module / infra

- **State isolated and protected**: own state key per environment, backend encrypted + locked,
  state access restricted (it contains secrets — see terraform-review references).
- **`prevent_destroy` + provider-level deletion protection** on stateful resources; a
  deliberate, documented path for when destruction is actually intended.
- **Plan output reviewed for replacements** on the promotion path — a module change that's
  in-place in dev can be `-/+` in prod (different resource arguments); prod plan review is a
  gate, not a formality.
- **Bootstrapping documented**: can this be stood up in an empty account/project from the repo
  alone? Circular dependencies (the bucket that holds the state of the stack that creates the
  bucket) resolved and written down.

## GitOps application

- **Promotion path exists** (dev → staging → prod as reviewed git operations, not editing the
  prod directory directly) and **rollback = git revert** — verified that a revert actually
  converges (no hooks/immutable fields blocking re-sync).
- **selfHeal/prune settings per environment are deliberate**: prune on in prod without
  deletion protection review = a path-rename away from mass deletion (see argocd-applicationset
  references for containment).
- **Secrets flow documented**: a new secret's journey (who creates it where, how it reaches the
  cluster) answerable by anyone on the team.
- **Bootstrap**: new cluster → running apps is a repo-driven procedure (app-of-apps or
  ApplicationSet), not tribal memory.

## Observability gate (any artifact)

Minimum bar before "production-ready" (deep-dive in observability-review skill):

- The three questions answerable from dashboards alone: is it up? is it erroring? is it slow?
- At least one page-severity alert on user-visible symptom, with runbook link, tested to
  actually route to a human (send a test alert through the real path once).
- Logs structured, with request/correlation IDs, retained long enough to debug last week's
  incident.
- On-call knows this service exists: escalation path + ownership recorded where on-call looks.

## Documentation gate

- README answers: what is it, who owns it, how to run locally, how to deploy, where are the
  dashboards/runbooks.
- The "bus test": someone other than the author has deployed and rolled it back at least once.
- Accepted risks written down (single-replica DB, no DR region, manual failover) — readiness
  review output is allowed to accept risk, not allowed to be silent about it.
