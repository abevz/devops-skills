---
name: cert-manager-debug
description: Use when debugging cert-manager certificate issues in Kubernetes — Certificate stuck not Ready, ACME challenge failures (HTTP-01/DNS-01), Let's Encrypt rate limits, renewal not happening, or cert-manager webhook errors. Mention "cert-manager", "certificate not ready", "acme challenge", "let's encrypt", "tls secret" as triggers.
license: MIT
compatibility: Works with Claude Code, Codex-style agents, CodeWhale, OpenCode, and other agents that support Agent Skills-style instructions.
---

# cert-manager Debug

## When to use

Use when a cert-manager-managed certificate misbehaves — a `Certificate` stuck not Ready, an
ACME challenge that never validates, TLS that expired even though cert-manager is installed, a
renewed cert the app still doesn't serve, or `kubectl apply` failing on cert-manager webhook
errors — and the cause needs to be found.

## Goal

Pinpoint which link in the issuance chain (Issuer → Certificate → CertificateRequest → Order →
Challenge → served TLS) is failing and why, backed by the resource's own status/events, and
propose a fix — without mutating live secrets, forcing renewals, or burning ACME rate limits.

## Workflow

1. **Baseline health first** — `kubectl get pods -n cert-manager`: controller, webhook, and
   cainjector all Running. A crashlooping webhook explains "nothing cert-manager related can
   even be applied"; a down controller explains "nothing progresses" with zero errors anywhere.
2. **Walk the chain top-down** — `kubectl describe` the `Certificate`, then its
   `CertificateRequest`, then `Order`, then `Challenge`(s). The deepest resource with an error
   in status/events names the culprit; quote that exact message. `cmctl status certificate
   <name> -n <ns>` walks the whole chain in one read-only command if `cmctl` is available.
3. **Issuer readiness** — `kubectl describe clusterissuer/<name>` (or `issuer`): is it Ready?
   ACME account registration failures (missing/invalid account key secret, unreachable ACME
   server) stall every certificate under that issuer at once.
4. **Route by challenge type** once a Challenge holds the error:
   - **HTTP-01**: does the `cm-acme-http-solver` pod and configured Ingress or Gateway
     HTTPRoute exist, and is
     `http://<domain>/.well-known/acme-challenge/<token>` reachable *from the internet*? The
     self-check runs from inside the cluster — hairpin-NAT homelab setups fail it even when the
     outside world would succeed.
   - **DNS-01**: is the `_acme-challenge.<domain>` TXT record visible at the *authoritative*
     nameservers? Provider credential errors and split-horizon DNS are the two classic causes.
5. **Check rate limits before any retry** — if the error mentions `rateLimited`, stop: deleting
   and recreating certificates makes it worse. Point experiments at the Let's Encrypt staging
   endpoint instead.
6. **Renewal and consumption issues** — cert renewed (secret updated, check
   `kubectl get certificate -o wide` and the secret's cert dates via openssl) but clients still
   see the old cert: the consuming workload loads TLS at startup only and needs a rollout;
   ingress controllers generally pick up secret changes, pods mounting the secret do not.
7. **Correlate and conclude** — tie the quoted status/event message to the specific chain link
   and cause; propose the minimal fix and how to verify issuance end-to-end afterwards.

## References

`references/playbooks.md` — chain-status and error-text → cause → fix tables (Certificate/
Order/Challenge states, HTTP-01 and DNS-01 walkthroughs, Let's Encrypt rate limits, webhook
apply failures, renewal/consumption issues). Route there as soon as the chain walk surfaces an
error message.

For reviewing TLS posture and secrets handling more broadly, see `kubernetes-security` and
`secrets-management`. For the ingress resource itself misrouting traffic (as opposed to the
certificate on it), start with `kubernetes-debug`.

## Safety rules

- Read-only first, always: `kubectl get/describe certificate,certificaterequest,order,challenge,
  issuer,clusterissuer`, `cmctl status certificate`, `dig`/`openssl s_client`. None of these
  mutate state.
- Never delete a Certificate's TLS secret or run `cmctl renew` as a diagnostic — deleting the
  secret takes the endpoint's TLS down until reissuance succeeds *and* counts against ACME rate
  limits. Propose it explicitly; the user confirms.
- Never suggest delete/recreate retry loops against Let's Encrypt production — the duplicate
  certificate limit (5/week per identical name set) can lock issuance for days. Experiments go
  to the staging endpoint via a *separate parallel* issuer.
- Do not switch an existing production Issuer/ClusterIssuer to the staging URL in place — every
  certificate under it would renew as untrusted.

## Output format

```
## Observations
<controller/webhook health, issuer readiness, from which command>

## Evidence
<chain walk: the deepest failing resource and its quoted status/event message>

## Root cause
## Fix
<manifest/config diff or action, marked as requiring user confirmation to apply>

## Verification
<certificate Ready check + end-to-end proof, e.g. openssl s_client -connect ... -servername ...>
```

## Quality checklist

- [ ] The full chain (Certificate → CertificateRequest → Order → Challenge) was walked and the
      deepest error message quoted, not paraphrased
- [ ] The challenge type was identified and its specific playbook followed (HTTP-01 external
      reachability / DNS-01 TXT at authoritative NS)
- [ ] Rate-limit implications were considered before proposing any delete/retry
- [ ] Only read-only commands were run without confirmation; secret deletion and forced renewal
      were marked as requiring it
- [ ] The fix includes an end-to-end verification command, not just "the Certificate is Ready"
