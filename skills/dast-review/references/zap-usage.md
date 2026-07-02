# ZAP usage patterns

Self-authored reference (no third-party source). Concrete ZAP invocations, auth, CI wiring, and
false-positive control for the DAST workflow. ZAP = the scanner formerly known as OWASP ZAP,
now maintained under Checkmarx; the `ghcr.io/zaproxy/zaproxy` images and `zaproxy/action-*`
Actions are the current distribution.

## The three scan scripts

| Script | Mode | What it does | Time |
|---|---|---|---|
| `zap-baseline.py` | Passive | Spiders the target, runs passive rules only (no attack payloads). Reports missing headers, cookie flags, info disclosure. | 1–5 min |
| `zap-full-scan.py` | Active | Baseline + active injection (SQLi, XSS, path traversal, etc.). Mutates state. | 20–60+ min |
| `zap-api-scan.py` | Active (API) | Imports an API definition and scans every endpoint; no crawling needed. | minutes |

```bash
# Passive PR gate (Docker)
docker run --rm -v "$(pwd):/zap/wrk:rw" -t ghcr.io/zaproxy/zaproxy:stable \
  zap-baseline.py -t https://staging.example.com -r report.html -w report.md \
  -c gen.conf                       # baseline config: known-accepted alerts
#  -I  → do not fail on warnings (report-only); omit -I to fail the job on findings

# Full active scan against STAGING (never prod)
docker run --rm -v "$(pwd):/zap/wrk:rw" -t ghcr.io/zaproxy/zaproxy:stable \
  zap-full-scan.py -t https://staging.example.com -r full.html

# API scan from an OpenAPI spec (best fit for Go/JSON backends)
docker run --rm -v "$(pwd):/zap/wrk:rw" -t ghcr.io/zaproxy/zaproxy:stable \
  zap-api-scan.py -t https://staging.example.com/openapi.json -f openapi -r api.html
#  -f accepts: openapi | soap | graphql
```

Exit codes: 0 = clean, 1 = failures (findings at fail threshold), 2 = warnings only, 3 = error.
The CI job's pass/fail hinges on this — don't `|| true` it away and call it a gate.

## Authentication — without it you scan the login page and nothing else

Modern ZAP: **Automation Framework** YAML is the maintainable path (replaces the old context-XML
+ env-var juggling). Sketch:

```yaml
# zap.yaml — run with:  zap.sh -cmd -autorun /zap/wrk/zap.yaml
env:
  contexts:
    - name: app
      urls: [https://staging.example.com]
      includePaths: ["https://staging.example.com.*"]
      excludePaths: ["https://staging.example.com/logout.*"]   # or the scan logs itself out
      authentication:
        method: browserBased        # or "json" / "form" / "http-header"
        parameters:
          loginPageUrl: https://staging.example.com/login
          loginPageWait: 5
      sessionManagement: {method: headerBasedSessionManagement}
      users:
        - name: scanner
          credentials: {username: scanner@example.com, password: "${SCAN_PW}"}   # env, never inline
      verification:
        method: response
        loggedInRegex: '\\Qlogout\\E'      # a string only present when authenticated
jobs:
  - {type: spider,  parameters: {context: app, user: scanner}}
  - {type: activeScan, parameters: {context: app, user: scanner}}
  - {type: report, parameters: {template: modern, reportFile: report}}
```

- Token/API auth (typical for a Go API): `method: http-header` injecting
  `Authorization: Bearer ${TOKEN}`, or a `httpsender` script to refresh short-lived tokens.
- The **`verification.loggedInRegex`/`loggedOutRegex` is load-bearing** — without a correct
  logged-in indicator, ZAP silently scans as anonymous and reports a falsely-clean result. Always
  confirm authenticated request count > 0 in the output.

## CI wiring (GitHub Actions)

```yaml
# passive gate on PRs — fails on NEW alerts vs the tracked baseline issue
- uses: zaproxy/action-baseline@v0.14.0
  with:
    target: https://staging.example.com
    rules_file_name: .zap/rules.tsv     # per-rule WARN/FAIL/IGNORE tuning
    cmd_options: '-a'                    # include alpha passive rules
    fail_action: true                   # make it an actual gate
# API scan variant: zaproxy/action-api-scan@v0.x with format: openapi
# full scan: zaproxy/action-full-scan@v0.x — schedule it, don't PR-gate it
```

`action-baseline` auto-maintains a GitHub issue as the living baseline: re-runs diff against it,
so the gate fires on *new* findings, not the standing set.

Placement rules:
- Passive baseline / API-scan-passive → PR, fail-on-High.
- Full active → nightly or merge-to-main, against staging, report-only into a queue.
- Never wire an active scan to a workflow that can target a production URL (parameterize the
  target and pin non-prod at the workflow level).

## False-positive control

DAST is noisy by design. Two mechanisms, use both:

- **Rules config** (`rules_file_name` TSV, or `-c gen.conf`): set each rule to `IGNORE`/`WARN`/
  `FAIL`. Downgrade the perennial informational ones (e.g. `10015` cache-control,
  `10096` timestamp disclosure) to WARN so they report without failing the gate.
- **Baseline issue / config file**: the accepted current-state set. Every entry is a decision —
  treat like a suppression: it needs a reason and periodic re-review (hand to
  `vulnerability-triage`). ❌ blanket-ignoring a whole rule class to make the gate green.

Fail threshold: gate on **High** confirmed findings; Medium after confirmation; Low/Info →
report only. A baseline gate failing on informational headers is disabled by the team within a
week — tune first, enforce second (same audit→enforce discipline as admission policy).

## API fuzzing companion (beyond ZAP)

For JSON APIs, ZAP's endpoint coverage + Schemathesis's property-based fuzzing complement:

```bash
schemathesis run --checks all --base-url https://staging.example.com https://staging.example.com/openapi.json
```

Finds contract violations, unexpected 500s, and malformed-input handling ZAP's payload set
doesn't emphasize. Spec completeness bounds coverage for both — an endpoint missing from the
OpenAPI spec is invisible to spec-driven scanning.

## Common failure modes

| Symptom | Cause |
|---|---|
| "Clean" scan, suspiciously fast | Not authenticated (scanned login page only) — check loggedInRegex and request count |
| Scan wanders to third-party domains | Context scope/includePaths too broad |
| Scanner logs out mid-scan | `/logout` not in excludePaths |
| Full scan corrupts staging data | Expected — active scans mutate; reseed staging, never point at real data |
| Gate flaps green/red across runs | No stable baseline; spider non-determinism — pin the baseline issue/config |
| API scan finds almost nothing | Incomplete OpenAPI spec; endpoints not in the definition aren't tested |
