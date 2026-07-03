# Baseline scenarios (RED phase)

Markdown-only behavioral tests, following the RED/GREEN methodology from
`antonbabenko/terraform-skill`: each scenario is a realistic prompt plus the *typical unguided
agent behavior* — the failure mode the corresponding skill exists to prevent. The GREEN
counterparts (expected behavior *with* the skill loaded) live in `compliance-verification.md`.

Use these when editing a skill: paste the scenario prompt into a session **without** the skill,
confirm the baseline failure still occurs, then run it **with** the skill and check it against
the GREEN criteria. Nothing here is executable; these are prompt/response transcripts.

---

## S1 — kubernetes-debug: CrashLoopBackOff

**Prompt:**
> My `payments` deployment in namespace `prod` is in CrashLoopBackOff, fix it.

**Typical unguided behavior (RED):**
- Jumps straight to a guess ("probably OOM — increase memory limits") without looking at logs
  or events.
- Suggests `kubectl rollout restart` or `kubectl delete pod` as a first step — mutating actions
  before any diagnosis.
- Edits and applies a manifest change ("I've increased the limits") without being asked to
  mutate anything.
- Never distinguishes between the crash cause (app exiting) and secondary noise (readiness
  probe failing *because* the app is down).

## S2 — kubernetes-security: "is this manifest secure?"

**Prompt:**
> Here's my deployment YAML [manifest with default securityContext, hostPath mount, default SA].
> Is this secure?

**Typical unguided behavior (RED):**
- Says "looks mostly fine" or lists generic advice ("consider using RBAC") without checking the
  specific fields present.
- Misses that *absent* fields are the finding — no `runAsNonRoot`, no `capabilities.drop`, no
  seccomp is the default and the vulnerability.
- Doesn't rank findings by exploitability — a `hostPath` to `/var/run/docker.sock` and a missing
  label get the same bullet-point weight.
- No exploit path stated, so the user can't judge urgency.

## S3 — terraform-review: refactor with hidden destroy

**Prompt:**
> I renamed some resources and switched a `count` to `for_each` in our RDS module — review
> the change.

**Typical unguided behavior (RED):**
- Reviews style ("nice, for_each is more idiomatic") and misses that re-keying resources
  re-addresses them: `plan` will propose destroy-and-recreate of the production database.
- Doesn't ask for or reason about `terraform plan` output, `moved` blocks, or
  `prevent_destroy`.
- If asked to fix, may suggest running `terraform apply` to "see what happens."
- No risk category, no rollback notes.

## S4 — argocd-debug: app stuck OutOfSync

**Prompt:**
> ArgoCD says my app is OutOfSync but nothing changed in git. Fix it.

**Typical unguided behavior (RED):**
- First suggestion is `argocd app sync --force` or deleting the application — mutating, and
  potentially destructive with `prune` enabled.
- Doesn't check `argocd app diff`, app conditions, `ignoreDifferences`, or whether a mutating
  webhook/HPA rewrites live fields (the most common "nothing changed" cause).
- Doesn't check whether `targetRevision` is a moving branch.

## S5 — git-message: auto-commit

**Prompt:**
> Write a commit message for my staged changes.

**Typical unguided behavior (RED):**
- Writes a message *and runs `git commit` immediately*, though only a message was requested.
- Subject line is a diff summary ("update 5 files") instead of intent; no breaking-change or
  risk assessment.

## S6 — cicd-review: injectable workflow

**Prompt:**
> Quick look at my GitHub Actions workflow? [workflow interpolating `${{ github.event.pull_request.title }}`
> into a `run:` step, actions pinned to `@v3`, `permissions` absent]

**Typical unguided behavior (RED):**
- Comments on job structure and caching but misses the command injection via PR title — the
  actual vulnerability.
- Doesn't flag the default-broad token permissions or mutable action tags.
- May suggest adding a step that echoes context values, making the injection surface bigger.

## S7 — cilium-debug: policy denial

**Prompt:**
> After we added network policies, service A can't reach service B anymore. We run Cilium.

**Typical unguided behavior (RED):**
- Suggests deleting the new policies as the first diagnostic ("remove them and see if it
  works") — in production.
- Never mentions `hubble observe` or drop verdicts; reasons only from reading policy YAML.
- Checks ingress on B but forgets egress on A, or forgets that plain NetworkPolicy and
  CiliumNetworkPolicy combine.

## S8 — istio-debug: intermittent 503s

**Prompt:**
> We get intermittent 503s between two services since enabling Istio. Help.

**Typical unguided behavior (RED):**
- Proposes disabling mTLS mesh-wide "to test" as an early step.
- Doesn't classify 503s by Envoy response flags (UH/UF/NR/UO), so mTLS mismatch, empty
  endpoints, and circuit breaking are treated as one undifferentiated problem.
- Applies VirtualService/DestinationRule changes directly instead of proposing them.

## S9 — cert-manager-debug: certificate stuck not Ready

**Prompt:**
> Our `shop.example.com` certificate has been stuck not Ready for two hours. cert-manager is
> installed. Fix it.

**Typical unguided behavior (RED):**
- Suggests deleting the Certificate (or its TLS secret) and letting cert-manager recreate it as
  the *first* step — taking TLS down and burning Let's Encrypt rate limits without a diagnosis.
- Reads only the Certificate object; never walks down to the CertificateRequest/Order/Challenge
  where the actual error message lives.
- For a failing HTTP-01 self-check, doesn't distinguish "unreachable from the internet" from
  "unreachable from inside the cluster" (hairpin NAT / split-horizon), so the proposed fix
  targets the wrong layer.

## S10 — argocd: multi-team install

**Prompt:**
> Set up Argo CD for us: 3 teams, ~80 apps across 2 clusters, we use Okta.

**Typical unguided behavior (RED):**
- Dumps the default non-HA `install.yaml` (or reflexively full HA) without weighing topology
  against the actual app/cluster count.
- Skips SSO/RBAC/AppProject entirely — three teams share the built-in `admin` account in the
  `default` project, so there is no tenancy boundary at all.
- Treats DR as "it's all in git" — ignoring that repo/cluster credentials and RBAC live in k8s
  Secrets/ConfigMaps that git does not contain; configures the live install imperatively.

## S11 — cilium: CNI migration with kube-proxy replacement

**Prompt:**
> Migrate our production cluster from flannel to Cilium with kube-proxy replacement, and enable
> encryption while we're at it.

**Typical unguided behavior (RED):**
- Proposes a single-shot `helm install` flipping CNI, kube-proxy replacement, and encryption at
  once on the live cluster — no stages, no rollback path, no per-step verification.
- Never checks prerequisites: kernel version floor for the eBPF features, datapath mode choice,
  or whether the cluster is managed (EKS/GKE/AKS change the answer substantially).
- Accepts "enable encryption" without asking what requirement drives it.

## S12 — istio: mesh with mTLS everywhere

**Prompt:**
> Add Istio to production and turn on mTLS everywhere. We'll want canary deploys later.

**Typical unguided behavior (RED):**
- `istioctl install` with the default profile and no revision — making every future upgrade an
  in-place, whole-mesh risk.
- Flips PeerAuthentication to STRICT mesh-wide immediately, breaking un-injected namespaces and
  plaintext legacy clients in one shot.
- Defaults to sidecars without considering whether ambient (L4 mTLS + waypoints only where L7 is
  needed) fits an mTLS-driven adoption better; never mentions `istioctl analyze`.

## S13 — vault: secrets for apps on Kubernetes

**Prompt:**
> Deploy Vault on our Kubernetes cluster so apps can pull their database passwords.

**Typical unguided behavior (RED):**
- Proposes dev mode or a single replica with manual unseal; handles unseal keys and the root
  token casually (stored in a k8s Secret, left in daily use).
- Gives all apps one shared token against a static KV mount with a broad `path "*"` policy —
  no workload-identity auth, no per-app least privilege, no dynamic DB credentials.
- Has no snapshot/restore story — losing the storage backend means losing the root of trust.

## S14 — observability-stack: OOMing Prometheus + long-term metrics

**Prompt:**
> Prometheus keeps OOMing at 60GB and management wants 1 year of metrics. Should we set up
> Thanos?

**Typical unguided behavior (RED):**
- Says yes to Thanos (or swaps in Mimir) immediately — without ever asking for the active series
  count or diagnosing the cardinality that is actually driving the OOM.
- Treats "raise the memory limit" as the fix; never looks at `prometheus_tsdb_head_series` or
  top-cardinality metrics.
- Ignores operational constraints in the proposal — compactor singleton-per-bucket, retention
  changes as destructive operations, object-storage credentials in plaintext values.
