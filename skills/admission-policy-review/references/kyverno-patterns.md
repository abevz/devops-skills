# Kyverno policy patterns

Self-authored reference (no third-party source). Working patterns for the policies the minimal
deny set requires. All examples start life with `validationFailureAction: Audit`.

**Version scope:** The `kyverno.io/v1` `ClusterPolicy` examples below use Kyverno's legacy
policy API. Kyverno v1.19 deprecates `ClusterPolicy`/`Policy` and says they will be removed in
v1.20. For v1.19 migrations and later releases, consult the corresponding CEL policy types
before copying these examples. This file has not been validated against a live Kyverno version.
[Kyverno upgrade guide](https://kyverno.io/docs/installation/upgrading/).

## Require digest, deny :latest / tag-only

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-image-digest
spec:
  validationFailureAction: Audit          # flip to Enforce after burn-down
  background: true                        # also report on existing resources
  rules:
    - name: require-digest
      match: {any: [{resources: {kinds: [Pod]}}]}
      validate:
        message: "Images must be referenced by digest (@sha256:...)."
        pattern:
          spec:
            containers:
              - image: "*@sha256:*"
```

Companion: restrict registries with `foreach` + `image: "registry.example.com/* | ghcr.io/org/*"`
pattern. Match on `Pod`, not Deployment — admission sees pods from every controller path.

## verifyImages — signature + attestation enforcement

```yaml
rules:
  - name: verify-signature
    match: {any: [{resources: {kinds: [Pod]}}]}
    verifyImages:
      - imageReferences: ["registry.example.com/*"]
        mutateDigest: true                # rewrites tag → digest on admit
        attestors:
          - entries:
              - keyless:
                  subject: "https://github.com/org/*/.github/workflows/release.yml@refs/tags/*"
                  issuer: "https://token.actions.githubusercontent.com"
                  rekor: {url: https://rekor.sigstore.dev}
        attestations:
          - type: https://cyclonedx.org/bom          # require SBOM attestation
            attestors: [...]
```

Gotchas that surface in review: Kyverno needs registry credentials (imagePullSecrets on its SA)
to fetch signatures from private registries; `mutateDigest: true` quietly fixes tag-deploys —
decide whether you want mutation or a hard deny; verifyImages counts as mutation → controllers
re-applying tag-based specs (GitOps!) will fight it unless git also moves to digests
(coordinate with `supply-chain-security`).

## Required resources and probes (what PSA can't do)

```yaml
- name: require-requests-limits
  match: {any: [{resources: {kinds: [Pod]}}]}
  validate:
    message: "CPU/memory requests and memory limit are required."
    pattern:
      spec:
        containers:
          - resources:
              requests: {memory: "?*", cpu: "?*"}
              limits: {memory: "?*"}          # deliberately NOT requiring cpu limit (throttling)
```

Note the deliberate omission of CPU limits — aligns with the reliability guidance in
kubernetes-yaml-review/references/reliability.md; a policy requiring CPU limits contradicts it.

## PolicyException with expiry (the only acceptable shape)

```yaml
apiVersion: kyverno.io/v2
kind: PolicyException
metadata:
  name: legacy-cron-no-digest
  namespace: batch-legacy
  annotations:
    owner: team-data
    reason: "vendor image, digest pinning blocked on vendor CI — tracking JIRA-1234"
    expires: "2026-09-01"
spec:
  exceptions:
    - policyName: require-image-digest
      ruleNames: [require-digest]
  match: {any: [{resources: {kinds: [Pod], namespaces: [batch-legacy], names: ["vendor-sync-*"]}}]}
```

Kyverno doesn't enforce `expires` natively — pair with a scheduled read-only report (below) or
a CI check over the exceptions directory that fails on past-due dates. Exceptions live in git
next to the policies, never created ad hoc via kubectl.

## Audit-mode burn-down queries (read-only)

```bash
# Violations by policy (cluster-wide)
kubectl get clusterpolicyreport -A -o json | jq -r '[.items[].results[]? | select(.result=="fail")] | group_by(.policy) | map({policy: .[0].policy, fails: length}) | sort_by(-.fails)'

# Who to notify: violations by namespace for one policy
kubectl get policyreport -A -o json | jq -r '.items[] | .metadata.namespace as $ns | .results[]? | select(.result=="fail" and .policy=="require-image-digest") | $ns' | sort | uniq -c | sort -rn

# Exceptions past their expiry annotation
kubectl get policyexceptions -A -o json | jq -r --arg today "$(date +%F)" '.items[] | select(.metadata.annotations.expires < $today) | .metadata.namespace + "/" + .metadata.name + " expired " + .metadata.annotations.expires'
```

## Rollout sequencing (per policy, not per engine)

1. Ship in `Audit` + `background: true` → PolicyReports populate for existing workloads.
2. Burn-down window with owners notified (queries above); track count → 0.
3. Flip `validationFailureAction: Enforce` in a PR — one policy at a time, prod last.
4. Alert on the webhook itself: kyverno pod availability and
   `kyverno_policy_results_total{result="fail"}` rate — enforcement silently degrading to
   Ignore-mode (failurePolicy) is the failure you won't see otherwise.
