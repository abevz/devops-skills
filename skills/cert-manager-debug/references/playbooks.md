# cert-manager chain playbooks

Walk `Certificate → CertificateRequest → Order → Challenge` with `kubectl describe` (or
`cmctl status certificate <name> -n <ns>` in one shot) and route by the deepest error message.
All commands read-only.

## Chain status → cause → fix

| Where / message | Cause | Diagnose / fix |
|---|---|---|
| Certificate `Ready=False`, `Issuing certificate as Secret does not exist` | Normal on *first* issuance | Only a problem if stuck >few minutes — go one level deeper |
| Certificate exists but no CertificateRequest appears | Controller down, or webhook rejecting internally | `kubectl -n cert-manager logs deploy/cert-manager` — the controller says why it won't act |
| No Certificate object at all despite ingress-shim annotation | Annotation wrong or missing: `cert-manager.io/cluster-issuer` vs `cert-manager.io/issuer`; typo'd issuer name | `kubectl describe ingress` events; the shim logs a reason when it skips an Ingress |
| CertificateRequest `Denied` | approver-policy (or similar) installed and no policy matches | `kubectl describe certificaterequest` shows the denial reason; fix the CertificateRequestPolicy, don't bypass it |
| Order: `no configured challenge solvers can be used for this challenge` | Issuer's solver `selector` (dnsZones/dnsNames) doesn't match the requested domain; wildcard names require a DNS-01 solver | Compare the Order's identifiers against every solver block in the (Cluster)Issuer |
| Issuer `Ready=False`, `secret not found` / ACME registration errors | ACME account key secret missing (common after restoring a cluster without that secret), or ACME server URL unreachable | `kubectl describe clusterissuer`; recreating the account happens automatically once the referenced secret name is writable — do not reuse a mismatched old key |
| `CAA record for <domain> prevents issuance` | Domain's CAA policy doesn't allow the CA | `dig CAA <domain> +short`; fix DNS, not cert-manager |
| Two Certificates reference the same `secretName` | They fight over the secret — endless reissue loop, rate-limit burn | `kubectl get certificate -A -o jsonpath` grep for the secretName; one of them must be renamed |

## HTTP-01 walkthrough

Challenge stuck `pending`, reason mentions the self-check:

1. **Which solver is configured?** Inspect the Issuer's `http01.ingress` or
   `http01.gatewayHTTPRoute` choice, then list the temporary solver pod and the matching
   route type: `kubectl get pods -n <ns> | grep cm-acme-http-solver`; for the Ingress solver,
   `kubectl get ingress -n <ns> | grep cm-acme-http-solver`; for the Gateway solver,
   `kubectl get httproute -n <ns> | grep cm-acme-http-solver`. The Gateway HTTP-01 solver is
   available from cert-manager 1.15. If a configured Gateway solver has no HTTPRoute, check
   that Gateway API CRDs are installed and cert-manager Gateway API support is enabled with
   `config.gatewayAPI.enabled: true` (Helm values). Some
   cert-manager components check for the CRDs only at startup; if the CRDs were installed
   later, plan a cert-manager Deployment restart before retrying. A missing solver resource
   also calls for controller logs; an existing one calls for route diagnostics.
2. **`wrong status code '404'`** — with an Ingress solver, check its class
   (`class`/`ingressClassName`) and path precedence. With a Gateway solver, check the
   temporary HTTPRoute's `parentRefs`, status conditions, and whether the referenced Gateway
   has an HTTP listener on port 80 that permits the Route's namespace. The solver must be
   reachable at `http://<domain>/.well-known/acme-challenge/<token>` — port 80, not a redirect
   to 443 that drops the path. [cert-manager HTTP-01 solvers](https://cert-manager.io/docs/configuration/acme/http01/).
3. **`failed to perform self check GET request` / connection refused / timeout** — the check
   runs *from inside the cluster* to the domain's public IP. Homelab signature: hairpin NAT —
   the outside world can reach the URL but the cluster cannot reach its own external IP. Verify
   from a true external vantage point before touching anything; split-horizon DNS pointing the
   domain at an internal IP inside the cluster has the same signature.
4. **Public reachability itself** — DNS A/AAAA record actually pointing at the LB (if
   external-dns manages it, has it created the record yet?), firewall allowing 80/tcp.

## DNS-01 walkthrough

1. **Is the TXT record there?** `dig TXT _acme-challenge.<domain> @<authoritative-ns> +short` —
   ask the *authoritative* nameserver, not your resolver. Record present there but
   `propagation check failed` persists = cert-manager's own check is querying a resolver with a
   different view (split-horizon): the `--dns01-recursive-nameservers` /
   `--dns01-recursive-nameservers-only` controller flags force it to public resolvers.
2. **Record never appears** — provider credentials: the Challenge's events quote the DNS
   provider API error verbatim (403s, wrong zone ID, missing permission). Fix the credential
   secret / scoping, don't retry blindly.
3. **`Could not find the SOA record`** — the zone cert-manager derived doesn't exist or is
   delegated oddly; for delegated `_acme-challenge` CNAMEs set `cnameStrategy: Follow` on the
   solver.
4. **Wildcards** — `*.example.com` and `example.com` are separate identifiers; both need DNS-01
   and both TXT records may exist simultaneously (two records at the same name is valid).

## Let's Encrypt rate limits

`urn:ietf:params:acme:error:rateLimited` in an Order/Challenge — stop retrying, route by which
limit:

- **Duplicate certificates: 5/week per identical name set.** The one people hit by
  delete/recreate loops. Waiting or changing the SAN set are the only outs.
- **Failed validations: 5/hour per hostname.** Hit while debugging a broken challenge — fix the
  challenge with the walkthroughs above *before* letting it retry.
- **Experiments belong on staging**: `https://acme-staging-v02.api.letsencrypt.org/directory`,
  as a *separate parallel* (Cluster)Issuer — never edit the production issuer's URL in place.

## Webhook apply failures

`kubectl apply` on any cert-manager resource fails with `failed calling webhook
"webhook.cert-manager.io"`:

- `connection refused` / `no endpoints available` — webhook pod not Ready:
  `kubectl get pods,endpoints -n cert-manager`; a NetworkPolicy blocking the API server →
  webhook path has the same signature.
- `x509: certificate signed by unknown authority` — cainjector hasn't injected the CA bundle
  into the webhook configurations: `kubectl -n cert-manager logs deploy/cert-manager-cainjector`
  and check the `ValidatingWebhookConfiguration`'s `caBundle` is non-empty.

## Renewal / consumption issues

- **Renewed but the app serves the old cert** — compare the secret's actual cert
  (`kubectl get secret <name> -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509
  -noout -dates`) with what's served (`openssl s_client -connect <host>:443 -servername <host>
  </dev/null 2>/dev/null | openssl x509 -noout -dates`). Secret new + served old = the consumer
  loads TLS at startup only: pods mounting the secret need a rollout (or a reloader); ingress
  controllers generally hot-reload secrets.
- **Renewal never attempted** — `kubectl get certificate -o wide` shows the renewal time;
  in the past with no action = controller down or `renewBefore` mis-set relative to `duration`.
- **Expired despite cert-manager installed** — often the Certificate was deleted but the secret
  lived on unmanaged; `cmctl status` on the secret's namespace shows whether anything owns it.
