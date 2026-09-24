# S15 — ingress: paired observation

Date: 2026-09-24. Model: baseline `claude-opus-5-5`, with skill `claude-opus-5-5`. Fresh Claude Code sessions; no cluster/infrastructure access or mutation was observed. Score uses [`GREEN criteria`](../../compliance-verification.md) verbatim as the rubric. The answers below are raw model output, not endorsed technical guidance; their factual claims were not independently verified.

## Prompt (identical in both arms)

> ingress-nginx is EOL and security wants it gone this quarter. We have ~40 Ingress objects. Move us to Gateway API.

## Harness

- Baseline: `claude -p --restricted --disable-slash-commands --tools '' --output-format json`.
- With skill: `claude -p --restricted --tools Read --allowedTools Read --add-dir /tmp/devops-skills-paired-candidate` and a system instruction to read the candidate skill catalog, then the applicable SKILL.md and references. The catalog contained the five candidates only. The runtime still advertised other MCP tools despite these flags; the trace shows no mutating tool calls.
- Skill visibly loaded: **yes**. Relevant successful Read trace: `/tmp/devops-skills-paired-candidate/skills/ingress/SKILL.md`, `/tmp/devops-skills-paired-candidate/skills/ingress/references/migration.md`, `/tmp/devops-skills-paired-candidate/skills/ingress/references/gateway-api.md`, `/tmp/devops-skills-paired-candidate/skills/ingress/references/ingress-nginx.md`.
- Candidate was assembled from PRs #2–#5 before merge: structure opening plus corrected references where applicable; S4 also has PR #3's final description. This tests a candidate snapshot, not automatic skill discovery from the installed directories.

## Score

| Arm | GREEN criteria | Result |
|---|---:|---|
| Without skill | 1/4 | observed |
| With skill | 2/4 | observed |

**FAIL.** The skill answer improved inventory and per-host rollout, but did not check a conformance report for the Cilium/Istio branches or reverify affinity and long-lived connections per host, so it misses strict GREEN criteria.

## Without skill: full answer

> I can't start the migration from this session. I have no shell or file access here, only Google Docs and Drive tools, so I can't read your manifests, run `kubectl`, or edit files. Also, `/home/abevz` has no commits, so it doesn't look like the repo that holds the Ingress objects.
>
> This is how I'd run it once I can reach the manifests and the cluster.
>
> ## Plan
>
> **1. Take stock of what you have (the key step).** Dump every Ingress and its annotations, e.g. `kubectl get ingress -A -o yaml`. Most Ingress objects convert cleanly. The work is in the nginx-specific annotations, so sort them into three groups:
>
> | Group | Common annotations | What they become in Gateway API |
> |---|---|---|
> | Direct replacement | `ssl-redirect`, `rewrite-target` (plain prefix), `backend-protocol: HTTPS/GRPC`, request and response headers | HTTPRoute `RequestRedirect` / `URLRewrite` / header filters, `BackendTLSPolicy`, `GRPCRoute` |
> | Depends on the controller you pick | `auth-url` (external auth), rate limits, session affinity, timeouts, body size, CORS, IP allowlists | That controller's own policy CRDs (Envoy Gateway `SecurityPolicy`/`BackendTrafficPolicy`, Istio, Cilium, etc.) |
> | No equivalent | `configuration-snippet`, `server-snippet`, regex rewrites with capture groups | Needs redesign. Security will also be happy these go: snippets were the root of the recent ingress-nginx CVEs. |
>
> **2. Pick a Gateway controller.** This sets how painful the "depends on the controller" group is. My default recommendation is **Envoy Gateway**: it covers auth, rate limits and CORS as first-class policies, and it's plain Envoy with no mesh required. Choose differently if you already run Istio or Cilium, or you're on GKE/EKS and want the cloud load balancer to be the Gateway.
>
> **3. Convert.** `ingress2gateway` (kubernetes-sigs) has an ingress-nginx provider that converts the Ingress objects plus the common annotations into Gateway and HTTPRoute objects. I'd use its output as a first draft only, then review every route from the second and third groups by hand. Set up one shared `Gateway` per load balancer, owned by the platform team. App teams then own their own `HTTPRoute`s, attached through `allowedRoutes`.
>
> **4. Check the tooling around it.**
> - cert-manager: move the certificate annotations onto the Gateway and enable Gateway API support.
> - external-dns: add `gateway-httproute` as a source.
> - Monitoring: dashboards and alerts built on the ingress-nginx metrics will stop working.
>
> **5. Run both side by side, then cut over one host at a time.** The new Gateway gets its own load balancer IP. Test each host with `curl --resolve`, then switch DNS for that host. Compare error rates and latency, and keep a rollback path by leaving the Ingress in place. With about 40 objects, a few batches per week gets you done well within the quarter.
>
> **6. Decommission.** Once the ingress-nginx access logs show no traffic for about a week, delete the Ingress objects, then uninstall the controller and remove its IngressClass and admission webhook.
>
> ## What I need from you
> 1. **Access.** Either run me somewhere with shell access in the repo that holds the manifests (and read-only `kubectl` if possible), or paste the output of `kubectl get ingress -A -o yaml`.
> 2. **How the manifests are managed.** Raw YAML, Helm, or Kustomize, and whether Argo CD or Flux deploys them.
> 3. **The platform.** Which cloud or on-prem, and whether you already run a service mesh or Cilium.
>
> Once I have the Ingress objects, I'll produce the annotation breakdown and a draft set of Gateway and HTTPRoute objects for the first batch.

## With skill: full answer

> I read the `ingress` skill and its references. I haven't looked at your cluster or manifests: I can't run commands in this session, and I don't know where your Ingress YAML lives. So this is the plan plus the inputs I need from you to turn it into real manifests. Nothing has been applied, and nothing should be applied as a single cutover.
>
> ## Where things stand
>
> - **ingress-nginx stopped getting updates in March 2026**, including security fixes. Security is right to want it gone. Until it is, every week it runs is an unpatched internet-facing proxy, so tighten it now (see "Before you start").
> - **What decides the timeline is how many of the ~40 Ingresses use raw nginx config**, not the 40 itself. Simple host/path/TLS rules convert almost automatically. `configuration-snippet` and `server-snippet` annotations can't be converted and each one has to be rebuilt by hand.
>
> ## Step 0: inventory (please run these and send me the output)
>
> ```bash
> # One line per Ingress: ns, name, class, hosts, nginx annotations in use
> kubectl get ingress -A -o json | jq -r '.items[] | [
>   .metadata.namespace, .metadata.name,
>   (.spec.ingressClassName // .metadata.annotations["kubernetes.io/ingress.class"] // "-"),
>   ([.spec.rules[]?.host] | join(",")),
>   ((.metadata.annotations // {}) | keys
>      | map(select(startswith("nginx.ingress.kubernetes.io/")) | sub("nginx.ingress.kubernetes.io/";""))
>      | join(","))
> ] | @tsv'
>
> # Blockers: raw nginx config snippets (count these first)
> kubectl get ingress -A -o json | jq -r '.items[]
>   | select((.metadata.annotations // {}) | keys | any(test("snippet")))
>   | "\(.metadata.namespace)/\(.metadata.name)"'
>
> # Global risky settings (snippets, ssl-passthrough, proxy-protocol)
> kubectl -n ingress-nginx get cm ingress-nginx-controller -o yaml
>
> # Which gateway to pick: CNI/mesh already running, and any Gateway API CRDs installed
> kubectl get pods -A -l k8s-app=cilium --no-headers | head -1
> kubectl get pods -A -l app=istiod --no-headers | head -1
> kubectl get crd | grep gateway.networking.k8s.io
> ```
>
> Please also tell me:
> 1. How TLS works today (cert-manager? which issuer? any wildcard certs?)
> 2. What's in front of ingress-nginx (cloud load balancer, MetalLB, hostPort?)
> 3. Where the Ingress manifests live in git (Argo CD, Flux, Helm charts?)
>
> Each Ingress then goes into one of three buckets:
>
> | Bucket | Typical annotations | Effort |
> |---|---|---|
> | Mechanical | host/path, TLS, `ssl-redirect`, `rewrite-target`, `canary-weight` | `ingress2gateway` output, then review |
> | Needs a feature check | `affinity`, `auth-url`/basic-auth, `proxy-body-size`, timeouts, `whitelist-source-range` | Gateway-specific settings; confirm the gateway supports each one before moving that host |
> | Blocker | `*-snippet` | Rebuild each one by hand or drop it; every one is also a security finding |
>
> ## Recommendation: which gateway to use
>
> - **Cilium is already your CNI** (with kube-proxy replacement): use Cilium's Gateway API support. There's no new proxy to run.
> - **Istio is already installed**: use Istio's Gateway API support.
> - **Neither**: use **Envoy Gateway**, or **nginx-gateway-fabric** if teams rely on nginx-specific behavior. In either case, check the project's published conformance report against the features from your inventory, especially cookie-based sticky sessions and external auth.
> - **Don't overbuild**: one gateway implementation and one shared Gateway (maybe two if you need separate public and internal load balancers). Don't give each team its own Gateway, and don't use gateway-specific settings where a standard Gateway API field does the job.
>
> ## Proposed layout (a skeleton for review; the class name depends on the choice above)
>
> The platform team owns the shared Gateway, and app teams own HTTPRoutes in their own namespaces:
>
> ```yaml
> apiVersion: gateway.networking.k8s.io/v1
> kind: Gateway
> metadata:
>   name: public
>   namespace: gateway-system
>   annotations:
>     cert-manager.io/cluster-issuer: <your-issuer>   # cert-manager needs its Gateway API support turned on
> spec:
>   gatewayClassName: <cilium|istio|eg|nginx>
>   listeners:
>     - name: http
>       port: 80
>       protocol: HTTP
>       allowedRoutes:
>         namespaces: { from: Same }        # only the HTTP→HTTPS redirect route attaches here
>     - name: https-example
>       port: 443
>       protocol: HTTPS
>       hostname: "*.example.com"
>       tls:
>         mode: Terminate
>         certificateRefs: [{ name: wildcard-example-com-tls }]
>       allowedRoutes:
>         namespaces:
>           from: Selector                 # the default "Same" silently ignores routes from app namespaces
>           selector:
>             matchLabels: { gateway-access/public: "true" }
> ---
> # Replaces ssl-redirect on all hosts
> apiVersion: gateway.networking.k8s.io/v1
> kind: HTTPRoute
> metadata: { name: https-redirect, namespace: gateway-system }
> spec:
>   parentRefs: [{ name: public, sectionName: http }]
>   rules:
>     - filters:
>         - type: RequestRedirect
>           requestRedirect: { scheme: https, statusCode: 301 }
> ---
> # Per app, in the team's namespace. Replaces one Ingress.
> apiVersion: gateway.networking.k8s.io/v1
> kind: HTTPRoute
> metadata: { name: shop, namespace: shop }
> spec:
>   parentRefs: [{ name: public, namespace: gateway-system, sectionName: https-example }]
>   hostnames: ["shop.example.com"]
>   rules:
>     - matches: [{ path: { type: PathPrefix, value: /api } }]
>       filters:                               # was: rewrite-target
>         - type: URLRewrite
>           urlRewrite: { path: { type: ReplacePrefixMatch, replacePrefixMatch: / } }
>       backendRefs:                           # was: second Ingress with canary-weight
>         - { name: shop-api, port: 8080, weight: 90 }
>         - { name: shop-api-canary, port: 8080, weight: 10 }
> ```
>
> A shared Gateway reads TLS secrets from its own namespace. Ingress-nginx read them from each app's namespace. So have cert-manager issue certs for the Gateway, or add ReferenceGrants. Don't copy secrets between namespaces.
>
> ## Rollout plan (about 14 weeks left in Q4)
>
> | Weeks | Work |
> |---|---|
> | 1 | Inventory and bucket all 40. Tighten ingress-nginx. Pick the gateway. Pin the Gateway API CRD and controller versions in git (install CRDs first). |
> | 2 | Run the new gateway alongside ingress-nginx on its **own load balancer/IP**, with a separate class name so the two controllers never claim the same objects. |
> | 3 | Pilot one low-risk host: write its HTTPRoute, test it against the new load balancer with `curl --resolve`, compare responses with the old path, then switch DNS with a short TTL. |
> | 4–9 | Move the mechanical bucket host by host, in batches of about 5 per week. **Rollback is reverting the DNS record**, so keep the old Ingress in git until the host has been confirmed for a few days. |
> | 6–11 | In parallel: the feature-check bucket, then blockers, with one pull request per rebuilt snippet host. Recheck anything that depends on client IP (allowlists, rate limits) after cutover. |
> | 12–13 | Delete the migrated Ingresses. Let long-lived websocket/SSE/gRPC connections drain from the old side. |
> | 14 | When `kubectl get ingress -A` is empty, remove the ingress-nginx controller, admission webhook, RBAC, IngressClass and load balancer. |
>
> **Before you start (this week):** set `allow-snippet-annotations: "false"` unless the inventory shows snippets are in use. Make sure the admission webhook is reachable only from the API server. Turn off `enable-ssl-passthrough` if nothing uses it.
>
> ## How to check each host
>
> ```bash
> kubectl get gateway,httproute -A
> kubectl describe gateway public -n gateway-system     # Accepted/Programmed, attachedRoutes count
> kubectl describe httproute <name> -n <ns>             # Accepted, ResolvedRefs (RefNotPermitted = missing ReferenceGrant)
> curl -sv --resolve shop.example.com:443:<new-LB-IP> https://shop.example.com/api/health
> ```
>
> Send me the inventory output and your answers to the three questions. From those I can bucket all 40, confirm the gateway choice, and draft the real Gateway/HTTPRoute manifests for review, starting with the pilot host.
