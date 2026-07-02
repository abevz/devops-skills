---
name: dast-review
description: Use when setting up or reviewing dynamic application security testing (DAST) for a web app or API — ZAP scan placement in CI/CD, authenticated scans, API scanning from an OpenAPI spec, and triaging DAST findings. Mention "DAST", "ZAP", "dynamic scan", "scan my API for vulns", "zap baseline" as triggers.
license: MIT
compatibility: Works with Claude Code, Codex-style agents, CodeWhale, OpenCode, and other agents that support Agent Skills-style instructions.
---

# DAST Review

## When to use

Use when adding dynamic security testing to a running web app or HTTP API, or reviewing an
existing DAST setup: where scans belong in the pipeline, how to authenticate them, how to scan
an API from its OpenAPI spec, and how to turn results into action. ZAP is the default engine;
the placement and triage logic apply to any DAST tool. Findings triage hands off to
`vulnerability-triage`; the pipeline wiring side is `cicd-review`.

## Goal

DAST that catches real runtime issues (injection, auth/session flaws, missing security headers,
exposed endpoints) without blocking PRs on slow active scans or pointing an attack tool at
production.

## Workflow

1. **Match scan type to pipeline stage** — the single most important decision:

   | Stage | Scan | Runtime | Blocks? |
   |---|---|---|---|
   | PR | ZAP **baseline** (passive: spider + passive rules, no attacks) | ~1–5 min | Yes — fast gate |
   | Merge to main / nightly | ZAP **full** (active: injects payloads) against **staging** | 20–60+ min | No — report + triage |
   | API change | ZAP **api-scan** from OpenAPI spec (+ Schemathesis for fuzzing) | minutes | PR-gate the passive part |
   | Scheduled | full active + auth against a dedicated scan environment | long | No |

2. **Never active-scan production** — active scans send real attack payloads (SQLi strings,
   stored-XSS, path traversal) that create/mutate/delete data and can DoS the target. Active
   goes against staging or an ephemeral environment seeded with disposable data. Passive
   (baseline) against prod is safe-ish but still generates load — prefer staging.

3. **Authenticate the scan or it tests almost nothing** — an unauthenticated scan only sees the
   login page. Configure auth (see references) so the scanner reaches the app behind login;
   verify it's actually authenticated (ZAP "Stats" / logged-in indicator) — a silently
   logged-out scan reports "clean" because it saw nothing.

4. **API-first for backend services** — for a Go/JSON API there's no HTML to crawl. Drive the
   scan from the contract: `zap-api-scan.py -t openapi.json -f openapi` imports every endpoint;
   pair with Schemathesis (`schemathesis run --checks all`) for property-based fuzzing that
   finds 500s/contract violations ZAP won't. The OpenAPI spec must be complete or coverage is
   only as good as the spec.

5. **Tune the context before trusting results** — scope (only in-scope hosts, or the scan
   wanders off to third parties), exclusions (logout URL — or the scanner logs itself out;
   destructive endpoints), and auth. An untuned scan is simultaneously noisy and low-coverage.

6. **Baseline the noise** — DAST is false-positive-prone. Establish an accepted baseline (ZAP
   `-c gen.conf` / the config file marking known-acceptable alerts) so the PR gate fails only on
   *new* findings, not the same 12 informational headers every run.

7. **Gate on severity, report the rest** — fail the PR only on High (and confirmed Medium);
   pass Low/Informational through as report annotations. A DAST gate that fails on
   informational findings gets disabled within a week.

8. **Triage findings** — route confirmed findings into `vulnerability-triage` (dedupe,
   reachability/exploitability, suppression-with-expiry). DAST-specific: confirm before
   escalating — reflected content isn't always exploitable XSS; a 500 isn't always a
   vulnerability. Reproduce the request/response the tool flagged.

## References

- `references/zap-usage.md` — baseline/full/api-scan invocation, auth configuration
  (form/JSON/OIDC/header), CI wiring (GitHub Action + raw Docker), false-positive baselining
  with the config file, and the report/gate pattern.

## Safety rules

- Only scan targets you are authorized to scan. Scanning systems you don't own or lack written
  permission for is illegal — confirm ownership/authorization before proposing any active scan.
- Never run an active scan against production or any environment with real user data — active
  payloads mutate and can destroy data. Propose staging/ephemeral targets.
- This skill designs and reviews scans; do not launch scans against a live target without the
  user's explicit go-ahead and confirmed authorization.
- Never echo captured secrets/session tokens from scan traffic in output — reference, don't quote.

## Output format

For setup/review:
```
## Scan placement (per stage: type, target env, gate/report)
## Authentication approach
## Scope & exclusions
## Findings gate policy (fail-on severity vs report)
## Proposed config/CI (ZAP invocation, for user review)
## Gaps
```

For triaging a scan report: hand off to `vulnerability-triage` format, adding a
"confirmed-reproducible?" note per finding.

## Quality checklist

- [ ] Active scans are targeted at staging/ephemeral, never production or real data
- [ ] Authorization to scan the target was confirmed
- [ ] The scan is authenticated (or its absence is a stated, deliberate limitation)
- [ ] API scans are driven from the OpenAPI spec, not blind crawling
- [ ] PR gate fails only on new High findings, not the informational baseline
- [ ] No scan was launched without explicit user go-ahead
