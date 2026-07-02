---
name: supply-chain-security
description: Use when designing or reviewing the artifact supply chain — SBOM generation, image signing, provenance/attestations, deploy-by-digest, and registry hygiene. Mention "supply chain", "sbom", "cosign", "sign images", "slsa", "deploy by digest" as triggers.
license: MIT
compatibility: Works with Claude Code, Codex-style agents, CodeWhale, OpenCode, and other agents that support Agent Skills-style instructions.
---

# Supply Chain Security

## When to use

Use when reviewing or designing the path from build to running artifact: SBOM, CVE scanning
placement, image signing and verification, provenance, registry policy, and digest-pinned
deploys. Complements `dockerfile-review` (image contents), `cicd-review` (pipeline as attack
surface), and `kubernetes-security` (cluster side).

## Goal

Make the deployed artifact traceable and verifiable end-to-end: know what was built, from what,
by whom — and make the cluster refuse anything that can't prove it.

## Workflow

1. **Inventory (SBOM)** — SBOM generated at build time (`syft <image> -o cyclonedx-json`),
   stored/attached as an attestation, not a CI log artifact that expires. SBOM from the built
   image beats SBOM from source (captures the base image and actual installed packages).
2. **Scan** — image CVE scan (`trivy image` / `grype`) gates the pipeline; scanning the SBOM
   (`grype sbom:./sbom.json`) lets you re-scan cheaply when new CVEs publish — without
   rebuilding. Check that *both* moments exist: on push AND continuously against the registry.
3. **Sign** — every image pushed to a prod-reachable registry is signed (`cosign sign`).
   Keyless (OIDC identity + Rekor transparency log) for public/CI builds; key-based (KMS) where
   keyless identity binding doesn't fit. Verify what the signature is bound to: the **digest**,
   never the tag.
4. **Attest** — SBOM and SLSA provenance attached as attestations (`cosign attest --predicate`)
   so verification policy can require them, not just a bare signature.
5. **Deploy by digest** — the GitOps repo references `image@sha256:...`, updated by the
   pipeline after sign/attest succeed. Tags are for humans; digests are for deploys. Check the
   whole chain: what the pipeline pushes = what git references = what admission verifies.
6. **Registry as boundary** — immutable tags enabled, push rights limited to CI identities
   (OIDC, not shared robot passwords), separate repos/paths for dev vs prod with promotion
   between them, retention that never deletes a digest currently referenced by git.
7. **Verify at admission** — cluster enforces the policy: signature + required attestations +
   allowed registry + digest-only. See `admission-policy-review` for the policy side.
8. **Gap-check the story** — walk one artifact backwards: running pod → digest → signature →
   provenance → source commit. Every broken link in that walk is a finding.

## References

- `references/signing-attestation.md` — cosign command patterns (keyless vs KMS), attestation
  predicate types, SLSA levels table, verification failure modes, and the digest-flow checklist.

## Safety rules

- This is a design/review skill: do not push, sign, or delete registry artifacts, and do not
  create/rotate signing keys — propose commands for the user.
- Never treat a signed image as "safe" — signing proves origin, not absence of vulnerabilities;
  keep the scan gate independent of the signature gate.
- Key material and OIDC trust configs are secrets-adjacent: reference where they live, never
  inline values.

## Output format

```
## Chain walk (pod → digest → signature → provenance → commit)
<which links exist, which are broken>

## Gaps by stage
- <stage> — <what's missing> — <risk> — <fix>

## Proposed pipeline additions
<concrete steps/commands, ordered, for user review>

## Verification policy requirements
<what admission should enforce once the chain exists>
```

## Quality checklist

- [ ] SBOM is generated from the built image and attached as an attestation
- [ ] Signatures are verified against digests, not tags
- [ ] The git → registry → admission digest chain was walked, not assumed
- [ ] Registry push rights and tag immutability were checked
- [ ] No artifact was pushed/signed/deleted by the agent
