# Signing, attestation, and provenance patterns

Self-authored reference (no third-party source). Command patterns and decision tables for the
supply-chain workflow.

## Keyless vs key-based signing

| | Keyless (Sigstore) | Key-based (KMS) |
|---|---|---|
| Identity | OIDC (CI workflow identity, e.g. GitHub Actions `sub` claim) | Whoever holds the key |
| Trust root | Fulcio CA + Rekor transparency log (public!) | Your KMS + its IAM |
| Revocation | Short-lived certs — nothing long-lived to leak | Key rotation procedure needed |
| Air-gapped / private | Needs private Sigstore stack (heavy) | Works anywhere |
| Fit | Public/SaaS CI, OSS | Regulated/air-gapped, existing KMS discipline |

Keyless caveat worth flagging in review: Rekor log entries are public — image names/digests of
private projects leak metadata unless you run a private instance.

```bash
# keyless (CI, OIDC ambient credentials)
cosign sign ghcr.io/org/app@sha256:<digest> --yes
# key-based via KMS
cosign sign --key awskms:///alias/cosign ghcr.io/org/app@sha256:<digest>
# verification pins WHO may have signed, not just "a signature exists":
cosign verify ghcr.io/org/app@sha256:<digest> \
  --certificate-identity-regexp '^https://github.com/org/app/\.github/workflows/release\.yml@refs/tags/v.*' \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com
```

- ❌ `cosign verify` without `--certificate-identity*` constraints — accepts anything Fulcio
  ever signed; the identity pin IS the policy.
- ❌ Signing tags (`cosign sign repo:v1.2`) — resolve to digest first; a re-pushed tag would
  inherit the old signature's trust.

## Attestations — signature vs statement

A bare signature says "this identity blessed this digest." Attestations carry typed claims:

| Predicate type | Claims | Generate |
|---|---|---|
| SLSA provenance | who/how/from-what built | `cosign attest --predicate prov.json --type slsaprovenance` (or the SLSA GitHub generator / GitHub artifact attestations) |
| CycloneDX / SPDX SBOM | what's inside | `syft <image> -o cyclonedx-json > sbom.json && cosign attest --predicate sbom.json --type cyclonedx` |
| Vulnerability scan | scan result at time T | `--type vuln` — useful for "was clean when shipped" audits |

Verification then requires the claim, not just a signature:
`cosign verify-attestation --type cyclonedx --certificate-identity-regexp ... <image@digest>`.
Admission engines (Kyverno `verifyImages.attestations`) can require predicates and even assert
on fields inside them (e.g. builder id, repo).

## SLSA levels — what each actually buys

| Level | Requirement | Blocks |
|---|---|---|
| L1 | Provenance exists | Honest-mistake provenance gaps; nothing adversarial |
| L2 | Provenance signed by the build platform | Tampering *after* the build |
| L3 | Build runs on hardened, isolated infra; provenance unforgeable by the build job itself | A compromised build *job* forging its own provenance |

Practical review stance: L2 via a hosted CI's native generator (GitHub artifact attestations /
slsa-github-generator) is the achievable target for most teams; claims of L3 deserve scrutiny
of the isolation story, not a checkbox.

## Digest-flow checklist (the chain that must not break)

1. CI builds → captures `IMAGE@DIGEST` from the push output (not from re-resolving the tag).
2. Sign + attest that digest.
3. GitOps repo bump commits the digest (Helm value / kustomize `newDigest` /
   `image: ...@sha256:`), authored by a pipeline identity whose commits are verifiable.
4. Admission verifies: allowed registry prefix, digest present (mutate-to-digest or deny tags),
   signature identity, required attestations.
5. Runtime audit: `kubectl get pods -A -o jsonpath='{..imageID}'` digests ⊆ signed digests.

Common breaks: image automation tools rewriting to tags; `latest` sneaking in via a sub-chart
default; retention deleting a digest still referenced by an old-but-active git revision
(rollback then fails — cross-check with `production-readiness` rollback gate).

## Verification failure modes (debugging `cosign verify`)

| Error | Usual cause |
|---|---|
| `no matching signatures` | Signed the tag not the digest; or image re-pushed (new digest); or wrong repo (signature lives beside the image) |
| `certificate identity does not match` | Workflow path/ref changed (renamed release.yml, tag vs branch ref) — update the pin deliberately, don't loosen to `.*` |
| `no matching attestations` | Attestation type mismatch (`--type` spelling), or attest step ran before the final push |
| Works locally, fails in admission | Admission engine lacks registry credentials to fetch signatures, or its trust root/Rekor config differs |
