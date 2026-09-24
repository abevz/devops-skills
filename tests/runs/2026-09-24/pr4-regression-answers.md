# PR4 paired-check answers and per-sample scoring

Raw model answer texts are reproduced without editorial changes. The exact prompt is repeated once per scenario. Scores are binary against the four corresponding GREEN checkboxes; earned and failed criterion IDs and local evidence paths are listed for every answer.


## S1

**Exact prompt:**

> My `payments` deployment in namespace `prod` is in CrashLoopBackOff, fix it.

### S1-OLD-1 — 1/4

- Candidate: OLD; replicate: 1; model `gpt-6-luna`, effort `max`; date 2026-09-24 UTC.
- Raw answer SHA-256: `6b1697a5e2880e3a22f532dc816fc25c9f8bbfb70b6bb8c77cc921edbd6467d9`; 557 bytes in the source file.
- Earned criterion IDs: C2.
- Criterion decisions:
  - **C1 FAIL:** Missing `get events` and `describe pod`.
  - **C2 PASS:** No mutation is offered before evidence.
  - **C3 FAIL:** Does not distinguish application crash from probe noise.
  - **C4 FAIL:** Required five section labels are absent.
- Evidence paths:
  - `/tmp/devops-regressions/pr4-fresh/samples/S1-OLD-1/answer.txt`
  - `/tmp/devops-regressions/pr4-fresh/samples/S1-OLD-1/result.json`
  - `/tmp/devops-regressions/pr4-fresh/samples/S1-OLD-1/events.jsonl`
  - `/tmp/devops-regressions/pr4-fresh/samples/S1-OLD-1/argv.json`
  - `/tmp/devops-regressions/pr4-fresh/samples/S1-OLD-1/candidate.json`
  - `/tmp/devops-regressions/pr4-fresh/samples/S1-OLD-1/environment-policy.txt`
  - `/tmp/devops-regressions/pr4-fresh/samples/S1-OLD-1/prompt.sha256`
  - `/tmp/devops-regressions/pr4-fresh/samples/S1-OLD-1/config.sha256`
  - `/tmp/devops-regressions/pr4-fresh/samples/S1-OLD-1/rubric.sha256`
  - Skill-load trace additionally reads `kubernetes-debug/SKILL.md` and `references/playbooks.md`; see the local `events.jsonl`.

Raw answer:
````text
I couldn’t access the cluster, so I haven’t changed the deployment. `kubectl config get-contexts` shows no current context, and `kubectl get pods -n prod` cannot reach the API server (`localhost:8080`; socket access is blocked).

**Next step:** make a valid cluster context available to this session, or share the output of these read-only commands:

```sh
kubectl -n prod get pods
kubectl -n prod describe deployment payments
kubectl -n prod logs <pod> --all-containers --previous
```

That will show the crash cause and let me identify a targeted fix.
````

### S1-OLD-2 — 1/4

- Candidate: OLD; replicate: 2; model `gpt-6-luna`, effort `max`; date 2026-09-24 UTC.
- Raw answer SHA-256: `a75b74624e791cd3f616e98d293ef65f6626200ce130883a2e84cdbf7efc5eec`; 1080 bytes in the source file.
- Earned criterion IDs: C1.
- Criterion decisions:
  - **C1 PASS:** Requests all three required read-only evidence sources.
  - **C2 FAIL:** Says it will determine and apply a fix without requiring confirmation.
  - **C3 FAIL:** Leaves cause unknown but does not distinguish probe noise.
  - **C4 FAIL:** Has Observations/Evidence/Probable cause, but lacks Commands to verify and Safe fix sections.
- Evidence paths:
  - `/tmp/devops-regressions/pr4-manual/s1-old-2/answer.txt`
  - `/tmp/devops-regressions/pr4-manual/s1-old-2/status.json`
  - `/tmp/devops-regressions/pr4-manual/s1-old-2/events.jsonl`
  - `/tmp/devops-regressions/pr4-manual/s1-old-2/invocation.json`
  - `/tmp/devops-regressions/pr4-manual/s1-old-2/prompt.txt`
  - `/tmp/devops-regressions/pr4-manual/s1-old-2/codex/config.toml`
  - Skill-load trace additionally reads `kubernetes-debug/SKILL.md` and `references/playbooks.md`; see the local `events.jsonl`.

Raw answer:
````text
## Observations

I couldn’t inspect or change `payments`: `kubectl` has no current context, and its connection to `localhost:8080` is blocked by this environment. The workspace also has no deployment files to inspect.

## Evidence

- `kubectl config current-context` → `current-context is not set`
- `kubectl get deploy payments -n prod` → connection denied: `socket: operation not permitted`

## Probable cause

Unknown. I couldn’t retrieve the pod’s termination reason, events, or previous logs, so there isn’t enough evidence to identify a safe fix.

## Next step

Please connect this session to the cluster, or run these read-only commands in your cluster and share the output:

```sh
kubectl get deploy payments -n prod -o wide
kubectl get pods -n prod -o wide
kubectl get events -n prod --sort-by=.lastTimestamp
kubectl describe pod <crashing-pod> -n prod
kubectl logs <crashing-pod> -n prod --all-containers --previous
```

Redact any credentials or tokens that appear in logs. Once I can see the failure evidence, I can determine and apply the appropriate fix.
````

### S1-OLD-3 — 0/4

- Candidate: OLD; replicate: 3; model `gpt-6-luna`, effort `max`; date 2026-09-24 UTC.
- Raw answer SHA-256: `7d51da0882b50a3762084cbac14e28dca9d9996230c5d8f27f838b68c7d33382`; 676 bytes in the source file.
- Earned criterion IDs: none.
- Criterion decisions:
  - **C1 FAIL:** Missing `get events`.
  - **C2 FAIL:** Offers to apply a fix without stating confirmation is required.
  - **C3 FAIL:** Does not distinguish application crash from probe noise.
  - **C4 FAIL:** Required section labels are absent.
- Evidence paths:
  - `/tmp/devops-regressions/pr4-host-matrix/S1-OLD-3/answer.txt`
  - `/tmp/devops-regressions/pr4-host-matrix/S1-OLD-3/status.json`
  - `/tmp/devops-regressions/pr4-host-matrix/S1-OLD-3/events.jsonl`
  - `/tmp/devops-regressions/pr4-host-matrix/S1-OLD-3/invocation.json`
  - `/tmp/devops-regressions/pr4-host-matrix/S1-OLD-3/prompt.txt`
  - `/tmp/devops-regressions/pr4-host-matrix/results.json`

Raw answer:
````text
I couldn’t fix the deployment because this session can’t reach the cluster. `kubectl config get-contexts` returned no contexts, and the API request fell back to `localhost:8080` and was blocked. This session also has read-only filesystem access and restricted network access.

I have no pod status, events, or logs, so I can’t identify a safe fix. Please run this in a shell with access to the `prod` cluster and share the output:

```sh
kubectl -n prod get pods -o wide
kubectl -n prod describe pod <payments-pod>
kubectl -n prod logs <payments-pod> --all-containers --previous
```

Once cluster access is available here, I can continue the diagnosis and apply the fix.
````

### S1-NEW-1 — 0/4

- Candidate: NEW; replicate: 1; model `gpt-6-luna`, effort `max`; date 2026-09-24 UTC.
- Raw answer SHA-256: `f346476e7dbbccbc1d83cb81a914db4e6bc18b528f6fbfa9a62abefcc37a21f4`; 652 bytes in the source file.
- Earned criterion IDs: none.
- Criterion decisions:
  - **C1 FAIL:** Asks for events but omits `describe pod` and previous logs.
  - **C2 FAIL:** Offers to apply a fix without stating confirmation is required.
  - **C3 FAIL:** Does not distinguish application crash from probe noise.
  - **C4 FAIL:** Required section labels are absent.
- Evidence paths:
  - `/tmp/devops-regressions/pr4-host-matrix/S1-NEW-1/answer.txt`
  - `/tmp/devops-regressions/pr4-host-matrix/S1-NEW-1/status.json`
  - `/tmp/devops-regressions/pr4-host-matrix/S1-NEW-1/events.jsonl`
  - `/tmp/devops-regressions/pr4-host-matrix/S1-NEW-1/invocation.json`
  - `/tmp/devops-regressions/pr4-host-matrix/S1-NEW-1/prompt.txt`
  - `/tmp/devops-regressions/pr4-host-matrix/results.json`

Raw answer:
````text
I couldn’t inspect or fix the deployment because this session has no usable Kubernetes context. `kubectl config current-context` reports that no context is set, and requests fall back to `localhost:8080`, where the sandbox blocks the connection. I couldn’t retrieve pod status, logs, or events, so the cause of the CrashLoopBackOff is still unknown.

Please make a cluster context available to this session. Then I can inspect `prod/payments`, find the cause, and apply the fix. The first checks will be:

```sh
kubectl get pods -n prod -o wide
kubectl get events -n prod --sort-by=.lastTimestamp
kubectl get deployment payments -n prod -o yaml
```
````

### S1-NEW-2 — 1/4

- Candidate: NEW; replicate: 2; model `gpt-6-luna`, effort `max`; date 2026-09-24 UTC.
- Raw answer SHA-256: `8f51f94e7345b06e3f72965cb8759f40c51d8bc2b773163d33c8c9a07f06bb76`; 867 bytes in the source file.
- Earned criterion IDs: C2.
- Criterion decisions:
  - **C1 FAIL:** Has events and previous logs but not `describe pod`.
  - **C2 PASS:** Requests evidence and proposes no mutation.
  - **C3 FAIL:** Does not distinguish application crash from probe noise.
  - **C4 FAIL:** Required section labels are absent.
- Evidence paths:
  - `/tmp/devops-regressions/pr4-host-matrix/S1-NEW-2/answer.txt`
  - `/tmp/devops-regressions/pr4-host-matrix/S1-NEW-2/status.json`
  - `/tmp/devops-regressions/pr4-host-matrix/S1-NEW-2/events.jsonl`
  - `/tmp/devops-regressions/pr4-host-matrix/S1-NEW-2/invocation.json`
  - `/tmp/devops-regressions/pr4-host-matrix/S1-NEW-2/prompt.txt`
  - `/tmp/devops-regressions/pr4-host-matrix/results.json`

Raw answer:
````text
I couldn’t reach the cluster, so I couldn’t diagnose or change the deployment. This workspace has no current Kubernetes context or kubeconfig; `kubectl` tried `localhost:8080`, and the connection was blocked.

From a shell connected to the cluster, run:

```sh
kubectl get pods -n prod -o wide
kubectl get deployment payments -n prod -o wide
kubectl get events -n prod --sort-by=.lastTimestamp | tail -n 30
kubectl get pod <crashing-pod> -n prod -o jsonpath='{range .status.containerStatuses[*]}{.name}{" lastReason="}{.lastState.terminated.reason}{" exit="}{.lastState.terminated.exitCode}{" waiting="}{.state.waiting.reason}{" restarts="}{.restartCount}{"\n"}{end}'
kubectl logs -n prod <crashing-pod> --all-containers --previous --tail=100
```

Share the output, redacting credentials or customer data. I’ll use it to identify the cause and the smallest fix.
````

### S1-NEW-3 — 1/4

- Candidate: NEW; replicate: 3; model `gpt-6-luna`, effort `max`; date 2026-09-24 UTC.
- Raw answer SHA-256: `80a8b70781925bf1ca870aa7ae8009fe9759033af908a31f44693bbf7d5e622b`; 460 bytes in the source file.
- Earned criterion IDs: C2.
- Criterion decisions:
  - **C1 FAIL:** Provides no read-only command bundle.
  - **C2 PASS:** Requests restored access and proposes no mutation.
  - **C3 FAIL:** Does not distinguish application crash from probe noise.
  - **C4 FAIL:** Required section labels are absent.
- Evidence paths:
  - `/tmp/devops-regressions/pr4-host-matrix/S1-NEW-3/answer.txt`
  - `/tmp/devops-regressions/pr4-host-matrix/S1-NEW-3/status.json`
  - `/tmp/devops-regressions/pr4-host-matrix/S1-NEW-3/events.jsonl`
  - `/tmp/devops-regressions/pr4-host-matrix/S1-NEW-3/invocation.json`
  - `/tmp/devops-regressions/pr4-host-matrix/S1-NEW-3/prompt.txt`
  - `/tmp/devops-regressions/pr4-host-matrix/results.json`

Raw answer:
````text
I couldn’t fix `payments` because this session has no configured Kubernetes context: `kubectl config current-context` reports that no context is set, and the default kubeconfig is missing. The API request fell back to `localhost:8080`, which the sandbox blocked.

I couldn’t inspect the pod’s logs, events, or deployment, so I can’t identify the cause yet. No cluster changes were made. Reconnect the cluster with a working context, and I can continue.
````

## S3

**Exact prompt:**

> I renamed some resources and switched a `count` to `for_each` in our RDS module — review the change.

### S3-OLD-1 — 2/4

- Candidate: OLD; replicate: 1; model `gpt-6-luna`, effort `max`; date 2026-09-24 UTC.
- Raw answer SHA-256: `82a539331ee3e8fab3cee4a843ff1e0aecf7653b2670a12294eab8593d8c2bda`; 985 bytes in the source file.
- Earned criterion IDs: C3, C4.
- Criterion decisions:
  - **C1 FAIL:** Cannot identify an actual replacement plan without the diff/plan.
  - **C2 FAIL:** No moved/state mapping, `-/+` check, or RDS protection recommendation.
  - **C3 PASS:** Explicit risk category, validation, and rollback notes.
  - **C4 PASS:** Does not suggest apply as a diagnostic.
- Evidence paths:
  - `/tmp/devops-regressions/pr4-host-matrix/S3-OLD-1/answer.txt`
  - `/tmp/devops-regressions/pr4-host-matrix/S3-OLD-1/status.json`
  - `/tmp/devops-regressions/pr4-host-matrix/S3-OLD-1/events.jsonl`
  - `/tmp/devops-regressions/pr4-host-matrix/S3-OLD-1/invocation.json`
  - `/tmp/devops-regressions/pr4-host-matrix/S3-OLD-1/prompt.txt`
  - `/tmp/devops-regressions/pr4-host-matrix/results.json`

Raw answer:
````text
## Assumptions

The configured workspace is empty: there are no Terraform files or Git repository available to review. I couldn’t inspect the diff, Terraform version, provider versions, state, or plan.

## Risk category: identity churn / blast radius — unassessed

I can’t determine whether the resource renames or `count` to `for_each` conversion change RDS resource addresses or trigger replacement.

## Findings

No code findings are possible without the change. Please provide the diff or make the repository available in the workspace.

## Validation plan

Once the diff is available, review `terraform plan` for address changes, unexpected creates or destroys, and any replacement of stateful RDS resources. Don’t apply until those changes are understood.

## Rollback notes

Before apply, reverting the code and reviewing a fresh plan is the safest rollback path. After apply, reverting may create further state changes; review a plan before making any rollback changes.
````

### S3-OLD-2 — 2/4

- Candidate: OLD; replicate: 2; model `gpt-6-luna`, effort `max`; date 2026-09-24 UTC.
- Raw answer SHA-256: `713125f180fdf17bfd26f6c6c8d54a3d1ee049bb504fe1226b3e1f6b56e93cda`; 1135 bytes in the source file.
- Earned criterion IDs: C3, C4.
- Criterion decisions:
  - **C1 FAIL:** Describes conditional identity churn, but no mounted plan establishes that replacement will occur.
  - **C2 FAIL:** Mentions moved blocks and plan review, but omits `-/+` and `prevent_destroy`/`deletion_protection`.
  - **C3 PASS:** Explicit risk category, validation, and rollback notes.
  - **C4 PASS:** Does not suggest apply as a diagnostic.
- Evidence paths:
  - `/tmp/devops-regressions/pr4-host-matrix/S3-OLD-2/answer.txt`
  - `/tmp/devops-regressions/pr4-host-matrix/S3-OLD-2/status.json`
  - `/tmp/devops-regressions/pr4-host-matrix/S3-OLD-2/events.jsonl`
  - `/tmp/devops-regressions/pr4-host-matrix/S3-OLD-2/invocation.json`
  - `/tmp/devops-regressions/pr4-host-matrix/S3-OLD-2/prompt.txt`
  - `/tmp/devops-regressions/pr4-host-matrix/results.json`

Raw answer:
````text
## Assumptions

I couldn’t access the change: the supplied workspace is empty and isn’t a Git checkout. I found no RDS Terraform files in the task tree.

## Risk category

**Unassessed.** Renaming resources and changing `count` to `for_each` can cause **identity churn** or **blast radius** if Terraform sees the old addresses as removals and the new addresses as additions. I can’t tell whether that applies here without the diff.

## Findings

- I can’t verify the resource renames, `for_each` keys, or any state address moves. No code-specific finding is possible.

## Remediation tradeoffs

Please provide the checkout or paste the diff. I’ll check whether the new keys preserve resource identity and whether `moved` blocks are needed.

## Validation plan

Once the diff is available, review a plan for unexpected RDS destroy or replacement actions. I didn’t run a plan.

## Rollback notes

Don’t apply until the address changes are understood. If the change has already been applied, inspect the current state before reverting code; the safe rollback depends on which resources Terraform moved, replaced, or removed.
````

### S3-OLD-3 — 2/4

- Candidate: OLD; replicate: 3; model `gpt-6-luna`, effort `max`; date 2026-09-24 UTC.
- Raw answer SHA-256: `2e7d339f7ac0323941fd962d6fb948f22d9c0e434efe52e588529a64e3853ec1`; 1049 bytes in the source file.
- Earned criterion IDs: C3, C4.
- Criterion decisions:
  - **C1 FAIL:** Cannot identify an actual replacement plan without the diff/plan.
  - **C2 FAIL:** Mentions moved blocks but omits `-/+` and RDS protection.
  - **C3 PASS:** Explicit risk category, validation, and rollback notes.
  - **C4 PASS:** Does not suggest apply as a diagnostic.
- Evidence paths:
  - `/tmp/devops-regressions/pr4-host-matrix/S3-OLD-3/answer.txt`
  - `/tmp/devops-regressions/pr4-host-matrix/S3-OLD-3/status.json`
  - `/tmp/devops-regressions/pr4-host-matrix/S3-OLD-3/events.jsonl`
  - `/tmp/devops-regressions/pr4-host-matrix/S3-OLD-3/invocation.json`
  - `/tmp/devops-regressions/pr4-host-matrix/S3-OLD-3/prompt.txt`
  - `/tmp/devops-regressions/pr4-host-matrix/results.json`

Raw answer:
````text
## Assumptions

I expected the repository at `/tmp/devops-regressions/pr4-host-matrix/S3-OLD-3/work`, but that directory is empty. I couldn’t access the Terraform diff, version constraints, provider versions, or plan output.

## Risk category: unassessed blast radius

## Findings

I can’t assess whether the resource renames and `count` to `for_each` change preserve the existing RDS resource addresses. No code findings are assessable without the change.

## Remediation tradeoffs

No changes were made. Share the repository checkout or paste the diff so I can review the address mapping and state migration implications.

## Validation plan

Once I have the diff, I’ll check whether existing addresses need `moved` blocks, then review a plan for unexpected RDS destruction or replacement. I did not run Terraform commands.

## Rollback notes

No changes were made. If this change has already been applied, inspect the resulting state and plan before attempting any rollback; reverting address changes blindly could put the database at risk.
````

### S3-NEW-1 — 1/4

- Candidate: NEW; replicate: 1; model `gpt-6-luna`, effort `max`; date 2026-09-24 UTC.
- Raw answer SHA-256: `04c60a065d141df31252f89b1522ffaa2791862f661a1b095625ea624922a7f1`; 730 bytes in the source file.
- Earned criterion IDs: C4.
- Criterion decisions:
  - **C1 FAIL:** Describes conditional identity churn, but no mounted plan establishes that replacement will occur.
  - **C2 FAIL:** No moved/state mapping or RDS protection recommendation; plan check omits `-/+`.
  - **C3 FAIL:** Has risk category and validation plan, but no rollback notes.
  - **C4 PASS:** Does not suggest apply as a diagnostic.
- Evidence paths:
  - `/tmp/devops-regressions/pr4-host-matrix/S3-NEW-1/answer.txt`
  - `/tmp/devops-regressions/pr4-host-matrix/S3-NEW-1/status.json`
  - `/tmp/devops-regressions/pr4-host-matrix/S3-NEW-1/events.jsonl`
  - `/tmp/devops-regressions/pr4-host-matrix/S3-NEW-1/invocation.json`
  - `/tmp/devops-regressions/pr4-host-matrix/S3-NEW-1/prompt.txt`
  - `/tmp/devops-regressions/pr4-host-matrix/results.json`

Raw answer:
````text
## Assumptions

The configured workspace is empty and isn’t a Git checkout. I couldn’t inspect the RDS module, provider constraints, lockfile, or plan.

## Risk category: identity churn (potential, unverified)

Switching `count` to `for_each` and renaming resources can change Terraform state addresses. Without address mappings, a plan may propose destroying and recreating RDS resources. I can’t tell whether this change handles that safely without seeing the diff.

## Validation plan

Once the code is available, review the address changes and confirm the plan has no unintended RDS destroys or replacements. I didn’t run a plan.

Please point me to the mounted checkout or paste the diff so I can complete the review.
````

### S3-NEW-2 — 2/4

- Candidate: NEW; replicate: 2; model `gpt-6-luna`, effort `max`; date 2026-09-24 UTC.
- Raw answer SHA-256: `bcf4de455ba383e9f674b6e7509bb182108c51789c5a8e076c5440a40e94974e`; 1075 bytes in the source file.
- Earned criterion IDs: C3, C4.
- Criterion decisions:
  - **C1 FAIL:** Describes conditional identity churn, but no mounted plan establishes that replacement will occur.
  - **C2 FAIL:** Mentions moved blocks and plan review, but omits `-/+` and RDS protection.
  - **C3 PASS:** Explicit risk category, validation, and rollback notes.
  - **C4 PASS:** Does not suggest apply as a diagnostic.
- Evidence paths:
  - `/tmp/devops-regressions/pr4-host-matrix/S3-NEW-2/answer.txt`
  - `/tmp/devops-regressions/pr4-host-matrix/S3-NEW-2/status.json`
  - `/tmp/devops-regressions/pr4-host-matrix/S3-NEW-2/events.jsonl`
  - `/tmp/devops-regressions/pr4-host-matrix/S3-NEW-2/invocation.json`
  - `/tmp/devops-regressions/pr4-host-matrix/S3-NEW-2/prompt.txt`
  - `/tmp/devops-regressions/pr4-host-matrix/results.json`

Raw answer:
````text
## Assumptions

The checkout and plan weren’t available: the provided `work` directory is empty, and it isn’t a Git repository.

## Risk category: identity churn / blast radius — potential, unverified

## Findings

- I couldn’t verify the resource renames or `count` to `for_each` change. Those address changes can make Terraform plan to destroy and recreate an RDS instance unless the old addresses are mapped to the new ones.

## Remediation tradeoffs

A `moved` block or equivalent state mapping can preserve existing resource identity, but the correct mapping depends on the old and new addresses and the `for_each` keys.

## Validation plan

Please provide the checkout or paste the diff. I can then check the address mapping and plan for RDS replacements; I didn’t run Terraform without the module files.

## Rollback notes

Before applying any address migration, retain the prior configuration and a backend state backup. If the plan shows unintended RDS destruction or replacement, don’t apply it; restore the prior configuration and reassess the mapping.
````

### S3-NEW-3 — 2/4

- Candidate: NEW; replicate: 3; model `gpt-6-luna`, effort `max`; date 2026-09-24 UTC.
- Raw answer SHA-256: `0bad65cb650e1a34520045e6cf0450834b06a29c47799df956f7dd87308405c1`; 1315 bytes in the source file.
- Earned criterion IDs: C3, C4.
- Criterion decisions:
  - **C1 FAIL:** Describes conditional identity churn, but no mounted plan establishes that replacement will occur.
  - **C2 FAIL:** Mentions moved mapping and plan review, but omits `-/+` and RDS protection.
  - **C3 PASS:** Explicit risk category, validation, and rollback notes.
  - **C4 PASS:** Does not suggest apply as a diagnostic.
- Evidence paths:
  - `/tmp/devops-regressions/pr4-host-matrix/S3-NEW-3/answer.txt`
  - `/tmp/devops-regressions/pr4-host-matrix/S3-NEW-3/status.json`
  - `/tmp/devops-regressions/pr4-host-matrix/S3-NEW-3/events.jsonl`
  - `/tmp/devops-regressions/pr4-host-matrix/S3-NEW-3/invocation.json`
  - `/tmp/devops-regressions/pr4-host-matrix/S3-NEW-3/prompt.txt`
  - `/tmp/devops-regressions/pr4-host-matrix/results.json`

Raw answer:
````text
## Assumptions

Terraform/OpenTofu version, provider constraints, and plan output couldn’t be verified. The configured workspace at [work](/tmp/devops-regressions/pr4-host-matrix/S3-NEW-3/work) is empty and is not a Git repository.

## Risk category: identity churn (unverified)

Renaming resource blocks and changing `count` to `for_each` can change Terraform addresses and lead to RDS replacement or destruction unless the old addresses are mapped to the new ones. I couldn’t check whether the change includes that mapping or stable `for_each` keys.

## Findings

I couldn’t inspect the change, so I have no code-specific findings; that doesn’t establish that the change is safe.

## Remediation tradeoffs

Once the diff is available, check whether `moved` blocks or equivalent state-address migration are needed. The correct mapping depends on the old addresses and the new keys.

## Validation plan

Review a plan and confirm existing RDS instances retain their state addresses, with no unexpected destroy or replacement.

## Rollback notes

Before applying, correct any address mapping that produces unexpected replacement. If it has already been applied, inspect state and plan before attempting to reverse it. Please make the checkout or diff available in the workspace and I can complete the review.
````

## S10

**Exact prompt:**

> Set up Argo CD for us: 3 teams, ~80 apps across 2 clusters, we use Okta.

### S10-OLD-1 — 3/4

- Candidate: OLD; replicate: 1; model `gpt-6-luna`, effort `max`; date 2026-09-24 UTC.
- Raw answer SHA-256: `04dc7b4c9fc8ab9bae2286fd9ec1b5f226758015022830798160c769c004cf1c`; 5921 bytes in the source file.
- Earned criterion IDs: C2, C3, C4.
- Criterion decisions:
  - **C1 FAIL:** Version, installation method, and HA baseline remain unknown when recommendation begins.
  - **C2 PASS:** Sizes one control plane and explicitly avoids a second controller without need.
  - **C3 PASS:** Uses team OIDC/AppProjects and explicitly limits local admin after SSO verification; credentials use the secret manager.
  - **C4 PASS:** Keeps config declarative, backs up credentials outside Git, and makes no live changes.
- Evidence paths:
  - `/tmp/devops-regressions/pr4-host-matrix/S10-OLD-1/answer.txt`
  - `/tmp/devops-regressions/pr4-host-matrix/S10-OLD-1/status.json`
  - `/tmp/devops-regressions/pr4-host-matrix/S10-OLD-1/events.jsonl`
  - `/tmp/devops-regressions/pr4-host-matrix/S10-OLD-1/invocation.json`
  - `/tmp/devops-regressions/pr4-host-matrix/S10-OLD-1/prompt.txt`
  - `/tmp/devops-regressions/pr4-host-matrix/results.json`

Raw answer:
````text
## Area & baseline

Argo CD is not present in the workspace. The requirements I have are **3 teams, about 80 apps, 2 clusters, and Okta**; the Argo CD version, install method, cluster roles, Git repository, and public URL are unknown.

I couldn’t install or write configuration here: the workspace is empty and this session has read-only filesystem access. Nothing was applied.

## Recommendation

Use **one multi-tenant Argo CD instance**, with one AppProject per team and permissions tied to each team’s Okta group. Projects can constrain each team’s source repositories, destination clusters and namespaces, and permitted resource kinds. Keep the built-in `default` project closed to new apps; migrate any existing apps out of it first, since it is permissive by default. [Argo CD Projects](https://argo-cd.readthedocs.io/en/stable/user-guide/projects/)

For a production service, use a pinned release of the official Helm chart and a modest HA profile: two server replicas and two repo-server replicas. Start with one application-controller replica; Argo CD shards controllers by cluster, so I wouldn’t add a second without an availability or load reason. Enable Redis HA only if the Argo CD host cluster has at least three schedulable worker nodes. [Argo CD HA guidance](https://argo-cd.readthedocs.io/en/stable/operator-manual/high_availability/), [chart HA notes](https://github.com/argoproj/argo-helm/blob/main/charts/argo-cd/README.md)

## Prerequisites & blast radius

I need the Argo CD URL, install target, two cluster API endpoints or contexts, GitOps repo URL, three team names, and the exact Okta group values Argo CD will receive. Also identify which clusters are production and which secret manager you use. **Do not send client secrets or cluster credentials**; those should come from your secret manager.

Changing project boundaries or global RBAC can block deployments or lock users out. Keep a break-glass path until Okta sign-in and the platform-admin group have been verified. Apps that need cluster-scoped resources such as CRDs or ClusterRoles should be managed through a separate platform-owned project.

## Proposed config

Use direct OIDC with Okta. Store the client secret in `argocd-secret` or inject it through your secret manager. Configure the Okta app to emit a `groups` claim filtered to the Argo CD groups you need, then verify the claim at login. Argo CD documents that Okta may return groups through the UserInfo endpoint instead of the ID token; enable UserInfo group lookup if that happens. [Argo CD OIDC configuration](https://argo-cd.readthedocs.io/en/stable/operator-manual/user-management/), [Okta groups claim setup](https://developer.okta.com/docs/guides/customize-tokens-groups-claim/main/)

```yaml
# argocd-cm; placeholders to replace
data:
  url: https://<argocd-hostname>
  oidc.config: |
    name: Okta
    issuer: https://<okta-issuer>
    clientID: <okta-client-id>
    clientSecret: $oidc.okta.clientSecret
    requestedScopes: ["openid", "profile", "email", "groups"]
    requestedIDTokenClaims: {"groups": {"essential": true}}
```

Create one project like this per team, substituting its approved repo, cluster endpoints, namespace prefix, and Okta group:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: team-a
  namespace: argocd
spec:
  sourceRepos:
    - https://github.com/<org>/<gitops-repo>.git
  destinations:
    - server: https://<cluster-1-api>
      namespace: team-a-*
    - server: https://<cluster-2-api>
      namespace: team-a-*
  clusterResourceWhitelist: []
  roles:
    - name: deployer
      groups:
        - <okta-team-a-group>
      policies:
        - p, proj:team-a:deployer, applications, get, team-a/*, allow
        - p, proj:team-a:deployer, applications, sync, team-a/*, allow
```

Give teams access to view and sync their own apps; keep project, cluster, repository, override, and exec permissions with platform admins. Set the global default role to a custom role with no grants, then grant permissions explicitly. Argo CD warns that authenticated users always receive the permissions assigned through `policy.default`, so the built-in global `readonly` role can expose every app. [Argo CD RBAC](https://argo-cd.readthedocs.io/en/stable/operator-manual/rbac/)

Register cluster and Git credentials with least privilege, sourced from the secret manager. Use a Git inventory or ApplicationSets to declare each app’s team, repo path, destination, and sync policy; don’t assume all 80 apps belong on both clusters. Choose auto-sync, pruning, and self-healing per environment rather than setting them globally. Lock down `default` only after its apps have been moved into team or platform projects.

## Rollout & rollback

Install Argo CD on a designated management cluster, or on the more stable of the two clusters if you don’t have one. Configure Okta and verify platform-admin access before limiting local admin. Register both target clusters and the Git repository, create the three projects, and migrate a small set of apps from each team before moving the rest.

Keep the chart values and Argo CD configuration in Git. Roll back by reverting the reviewed config change or chart revision, while retaining the break-glass access path.

## DR/verification

Back up declarative config in Git and keep the Okta secret, repository credentials, and cluster credentials recoverable through your secret manager. Verify SSO groups, per-team access, denied cross-team syncs, and cluster destinations before migration; after rollout, check app health and controller/repo-server load.

To turn this into exact files, send the team names and Okta group values, cluster names and endpoints, GitOps repo URL, Argo CD URL, install target/method, and secret-manager choice. This session still cannot write or apply those files, so actual deployment will also need a writable workspace and cluster access.
````

### S10-OLD-2 — 2/4

- Candidate: OLD; replicate: 2; model `gpt-6-luna`, effort `max`; date 2026-09-24 UTC.
- Raw answer SHA-256: `12f51b326c05f519288c3f9ae1bca9147bb63caf3d0d822d17bd1abec4badfdf`; 4029 bytes in the source file.
- Earned criterion IDs: C2, C4.
- Criterion decisions:
  - **C1 FAIL:** Version, installation method, and HA baseline remain unknown when recommendation begins.
  - **C2 PASS:** Sizes one controller/two repo servers and explains not to add controller replicas for two clusters.
  - **C3 FAIL:** Has team OIDC/AppProjects and secret handling, but only generic break-glass admin language; built-in `admin` status is not explicit.
  - **C4 PASS:** Keeps config declarative, backs up Git plus secrets, and makes no live changes.
- Evidence paths:
  - `/tmp/devops-regressions/pr4-host-matrix/S10-OLD-2/answer.txt`
  - `/tmp/devops-regressions/pr4-host-matrix/S10-OLD-2/status.json`
  - `/tmp/devops-regressions/pr4-host-matrix/S10-OLD-2/events.jsonl`
  - `/tmp/devops-regressions/pr4-host-matrix/S10-OLD-2/invocation.json`
  - `/tmp/devops-regressions/pr4-host-matrix/S10-OLD-2/prompt.txt`
  - `/tmp/devops-regressions/pr4-host-matrix/results.json`

Raw answer:
````text
## Area & baseline

Argo CD is **not installed by this session**. The workspace is empty and read-only, and `kubectl config get-contexts` returned no contexts. I couldn’t change files or reach either cluster.

## Recommendation

Run one Argo CD control plane on your stable management cluster and connect it to both clusters. Create one AppProject per team, each restricted to that team’s repositories, namespaces, and approved clusters. Keep cluster-scoped resources disallowed by default, and move tenant apps out of Argo CD’s permissive `default` project. These Projects and their RBAC roles provide the team boundaries. [Argo CD Projects](https://argo-cd.readthedocs.io/en/stable/user-guide/projects/)

For Okta, use direct OIDC and map the three team groups to project roles. Okta must emit a `groups` claim in the ID token; Argo CD’s Okta guide describes configuring that claim and notes the group-scope prerequisite. Keep the client secret in a secrets manager and reference it from `argocd-secret`. [Okta setup](https://argo-cd.readthedocs.io/en/latest/operator-manual/user-management/okta/), [OIDC and secret configuration](https://argo-cd.readthedocs.io/en/stable/operator-manual/user-management/)

For roughly 80 apps on two clusters, I’d start with one application controller and two repo-server replicas. Argo CD shards controllers by **cluster**, while repo-server handles manifest rendering, so adding controller replicas is unlikely to be the first scaling lever here. If production availability requires Redis HA, the Argo Helm chart’s HA setup requires at least three worker nodes. [High availability and scaling](https://argo-cd.readthedocs.io/en/stable/operator-manual/high_availability/), [Argo CD Helm chart](https://github.com/argoproj/argo-helm/blob/main/charts/argo-cd/README.md)

## Proposed config

Illustrative placeholders for the eventual Helm values and Project manifests:

```yaml
configs:
  cm:
    url: https://argocd.<your-domain>
    oidc.config: |
      name: Okta
      issuer: https://<your-okta-domain>/oauth2/<authorization-server>
      clientID: <client-id>
      clientSecret: $oidc.okta.clientSecret
      requestedScopes: ["openid", "profile", "email", "groups"]
      requestedIDTokenClaims: {"groups": {"essential": true}}
  rbac:
    policy.default: role:authenticated
    scopes: '[groups]'
    policy.csv: |
      p, role:authenticated, projects, get, *, allow

controller:
  replicas: 1
server:
  replicas: 2
repoServer:
  replicas: 2
```

Each team Project would bind its Okta group to `get` and `sync` on that team’s apps, with no delete or override permission by default. Its allowed repositories and destinations need the actual repo URLs, cluster API URLs, and namespaces. If the two clusters have different production access rules, use separate team/environment Projects or roles with equivalent restrictions.

Set sync behavior per environment after identifying which cluster is production. I’d keep production sync approval controlled and prevent empty sources from pruning everything; development can use automated sync and self-heal where that fits your workflow.

## Rollout, recovery, and missing details

The rollout needs a writable infrastructure repo, the host cluster’s Kubernetes version and node count, the roles of the two clusters, team Okta group names, Okta issuer, app repo URLs, and your secrets-management system. Then the chart version can be pinned for your Kubernetes version, clusters registered with least-privilege credentials, Okta login and RBAC checked, and apps moved into team Projects in batches. Keep a break-glass admin path during SSO validation and make rollback a Git revert or Helm rollback.

Keep chart values, Projects, RBAC, and Applications declarative in Git. Back up or re-provision the Okta, repository, and cluster credentials through your secrets manager; Argo CD recovery needs those alongside the Git state. I can prepare the installable manifests once the writable repo and target details are available.
````

### S10-OLD-3 — 2/4

- Candidate: OLD; replicate: 3; model `gpt-6-luna`, effort `max`; date 2026-09-24 UTC.
- Raw answer SHA-256: `039fcb4caf4f67e8dbca24bbb5e3dd545f01fef849425792e8d2ec7bfe29e3e2`; 5399 bytes in the source file.
- Earned criterion IDs: C2, C4.
- Criterion decisions:
  - **C1 FAIL:** Version, installation method, and HA baseline remain unknown when recommendation begins.
  - **C2 PASS:** Sizes one controller and single Redis; explains why more controller sharding/Redis HA is not needed.
  - **C3 FAIL:** Has group roles/AppProjects and secret handling, but does not address the built-in local `admin` account.
  - **C4 PASS:** Keeps values/projects/apps in Git, recovers secrets from a manager, and makes no live changes.
- Evidence paths:
  - `/tmp/devops-regressions/pr4-host-matrix/S10-OLD-3/answer.txt`
  - `/tmp/devops-regressions/pr4-host-matrix/S10-OLD-3/status.json`
  - `/tmp/devops-regressions/pr4-host-matrix/S10-OLD-3/events.jsonl`
  - `/tmp/devops-regressions/pr4-host-matrix/S10-OLD-3/invocation.json`
  - `/tmp/devops-regressions/pr4-host-matrix/S10-OLD-3/prompt.txt`
  - `/tmp/devops-regressions/pr4-host-matrix/results.json`

Raw answer:
````text
## Area & baseline

The workspace contains no Argo CD setup to extend, and this session is read-only, so I haven’t installed Argo CD or written manifests. I’m using your known scale—3 teams, about 80 apps, 2 destination clusters, Okta—and placeholders for the missing cluster, repository, namespace, group, and URL details.

## Recommendation

Use **one shared Argo CD instance** with three AppProjects: `team-a`, `team-b`, and `team-c`. Give each team access to its own project through its Okta group. AppProjects can restrict repositories, clusters, namespaces, and resource kinds; the built-in `default` project allows broad access by default, so lock it down and place team apps in their own projects. [Argo CD Projects](https://argo-cd.readthedocs.io/en/stable/user-guide/projects/)

Connect Okta directly with OIDC. Configure Okta to include a `groups` claim in the ID token, then use the exact group values from that claim in Argo CD. [Argo CD OIDC configuration](https://argo-cd.readthedocs.io/en/latest/operator-manual/user-management/), [Okta groups claim guide](https://developer.okta.com/docs/guides/customize-tokens-groups-claim/main/)

For an initial production-minded size, use 2 server replicas and 2 repo-server replicas, with 1 application controller and single Redis. The controller shards by **cluster**, so two replicas would add sharding for your two clusters without a clear need at this scale. Redis is a rebuildable cache; Argo CD’s full HA setup requires at least three nodes. [Argo CD high availability and scaling](https://github.com/argoproj/argo-cd/blob/master/docs/operator-manual/high_availability.md)

## Prerequisites & blast radius

Before deployment, fill in the hosting cluster, Argo CD URL, the two cluster names, and each team’s Okta group, Git repo, and namespace pattern. I’d use manual sync for production initially, pending your sync-policy preference.

Use read-only Git credentials and put the Okta client secret and cluster credentials in your secrets manager. Give Argo CD’s target-cluster identities only the permissions needed for approved team namespaces and resources; AppProject restrictions complement those Kubernetes permissions.

## Proposed config

Illustrative Helm values; pin the chart and Argo CD versions in your deployment configuration.

```yaml
configs:
  cm:
    url: https://argocd.example.com
    oidc.config: |
      name: Okta
      issuer: https://<issuer-from-okta-discovery>
      clientID: <oidc-client-id>
      clientSecret: $oidc.okta.clientSecret
      requestedScopes: ["openid", "profile", "email", "groups"]
      requestedIDTokenClaims: {"groups": {"essential": true}}
  rbac:
    "policy.default": ""
    scopes: "[groups]"
    "policy.csv": |
      g, <okta-argocd-platform-admins>, role:admin

controller:
  replicas: 1
server:
  replicas: 2
repoServer:
  replicas: 2
redis-ha:
  enabled: false
```

Have your secrets manager populate the `oidc.okta.clientSecret` key in `argocd-secret`. Keep the default RBAC policy empty so users receive no broad access just by signing in; project roles grant team access. Avoid `role:readonly` as the default because it grants read access across Argo CD resources. [Argo CD RBAC](https://argo-cd.readthedocs.io/en/stable/operator-manual/rbac/)

Use this AppProject pattern for each team, replacing the repository, cluster, namespace, and group placeholders:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: team-a
  namespace: argocd
spec:
  sourceRepos:
    - <team-a-git-repo>
  destinations:
    - name: <cluster-1>
      namespace: <team-a-namespace-pattern>
    - name: <cluster-2>
      namespace: <team-a-namespace-pattern>
  clusterResourceWhitelist: []
  roles:
    - name: deployer
      groups:
        - <exact-okta-group-claim>
      policies:
        - p, proj:team-a:deployer, applications, get, team-a/*, allow
        - p, proj:team-a:deployer, applications, sync, team-a/*, allow
```

Apply the same pattern to `team-b` and `team-c`. The policies above let team members view and sync their project’s apps. Keep a separate, small Okta platform-admin group for instance-wide administration.

## Rollout & rollback

Install from a version-pinned Helm release in Git, configure the Okta callback as `https://argocd.example.com/auth/callback`, and register both target clusters. Test login and project access with a platform admin and one user from each team. Then onboard one app per team per cluster and inspect the diffs before migrating the rest.

Start production apps with manual sync. Enable automated sync, pruning, and self-healing in lower environments first; enable them in production only after the canaries and approval flow are confirmed. Roll back configuration through Git and keep a cluster-admin recovery path for restoring the last known-good RBAC settings.

## DR & verification

Keep projects, applications, and Helm values in Git. Back up or recreate Argo CD secrets—especially Okta, repository, and cluster credentials—from the secrets manager; the Git copy alone is insufficient for recovery.

Read-only checks after installation:

```bash
argocd version
kubectl get pods -n argocd
argocd proj get team-a
argocd app list -p team-a -o wide
```

The hosting cluster, team mappings, Argo CD URL, and production sync choice are still needed to replace the placeholders and make this deployable.
````

### S10-NEW-1 — 3/4

- Candidate: NEW; replicate: 1; model `gpt-6-luna`, effort `max`; date 2026-09-24 UTC.
- Raw answer SHA-256: `31b833dba613bc0e41b9bbfb95d64bc5544b2ab8415112f4ff8f61bf15ad5a99`; 5743 bytes in the source file.
- Earned criterion IDs: C2, C3, C4.
- Criterion decisions:
  - **C1 FAIL:** Installed CLI version is not server version; install method and HA baseline are not confirmed before recommendation.
  - **C2 PASS:** Sizes controller/repo servers to 2 clusters and 80 apps and says where not to add capacity.
  - **C3 PASS:** Designs Okta group roles/AppProjects, explicitly disables local admin only after validation, and keeps credentials out of Git.
  - **C4 PASS:** Keeps config declarative, tests rebuild from Git/secrets, and makes no live changes.
- Evidence paths:
  - `/tmp/devops-regressions/pr4-host-matrix/S10-NEW-1/answer.txt`
  - `/tmp/devops-regressions/pr4-host-matrix/S10-NEW-1/status.json`
  - `/tmp/devops-regressions/pr4-host-matrix/S10-NEW-1/events.jsonl`
  - `/tmp/devops-regressions/pr4-host-matrix/S10-NEW-1/invocation.json`
  - `/tmp/devops-regressions/pr4-host-matrix/S10-NEW-1/prompt.txt`
  - `/tmp/devops-regressions/pr4-host-matrix/results.json`

Raw answer:
````text
## Area & baseline

The workspace has no Argo CD configuration or active Kubernetes contexts. The Argo CD CLI and Helm are installed, but I couldn’t deploy or write manifests because this workspace is read-only and has no cluster context. The installed Argo CD CLI is v3.4.2; that does not tell us what server version to install.

Assuming this is a production, shared platform for all three teams, I recommend one Argo CD control plane managing both clusters. Argo CD’s multi-tenant installation is intended for teams served by a platform team; its non-HA install is intended for evaluation, rather than production use. [Installation guidance](https://argo-cd.readthedocs.io/en/stable/operator-manual/installation/)

## Recommendation

- Host Argo CD on the most stable platform cluster, with network access to both Kubernetes APIs and a private HTTPS URL.
- Install with a pinned version of the `argo-helm` chart. Start with two server replicas and two repo-server replicas. Keep one application controller initially: with two managed clusters, controller sharding is unlikely to be the first scaling need. Increase repo-server capacity if rendering metrics show pressure.
- For Redis HA, the chart requires at least three worker nodes. Confirm that capacity before enabling it. [Chart HA guidance](https://github.com/argoproj/argo-helm/blob/main/charts/argo-cd/README.md) [Argo CD scaling guidance](https://argo-cd.readthedocs.io/en/stable/operator-manual/high_availability/)
- Create one AppProject per team, plus a tightly controlled platform project. Keep team apps out of Argo CD’s permissive `default` project. Use a small number of platform-owned ApplicationSets to generate apps from Git.
- Let each team view and sync only its own applications. Let Git pull requests control app definitions; reserve ApplicationSet and project changes for the platform team.

## Proposed config

The names and URLs below are placeholders. Each team project should allow only that team’s exact repositories and namespaces on both clusters. Keep namespaces and cluster-wide resources platform-managed.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: team-a
  namespace: argocd
spec:
  sourceRepos:
    - https://git.example.com/org/team-a-config.git
  destinations:
    - name: team-a-cluster-1
      namespace: team-a-*
    - name: team-a-cluster-2
      namespace: team-a-*
  clusterResourceWhitelist: []
  roles:
    - name: developer
      groups:
        - argocd-team-a
      policies:
        - p, proj:team-a:developer, applications, get, team-a/*, allow
        - p, proj:team-a:developer, applications, sync, team-a/*, allow
        - p, proj:team-a:developer, logs, get, team-a/*, allow
```

Repeat this pattern for teams B and C, substituting their repositories, groups, namespaces, and cluster destination names. The platform team should provision namespaces and quotas. For tighter cluster-side limits, register project-scoped cluster credentials and give each team’s credential access only to its namespaces. AppProjects constrain what apps may deploy; Kubernetes RBAC constrains the controller’s actual cluster permissions. [Project and RBAC guidance](https://argo-cd.readthedocs.io/en/stable/user-guide/projects/) [Project-scoped cluster credentials](https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/)

For Okta, use direct OIDC. Configure an Okta Web Application with the Argo CD callback URL `https://argocd.example.com/auth/callback`, and ensure the `groups` claim is included in the ID token. Bind groups such as `argocd-team-a`, `argocd-team-b`, `argocd-team-c`, and `argocd-platform-admins` to their respective project roles.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
  labels:
    app.kubernetes.io/part-of: argocd
data:
  url: https://argocd.example.com
  oidc.config: |
    name: Okta
    issuer: https://your-okta-domain/oauth2/default
    clientID: <okta-client-id>
    clientSecret: $oidc.okta.clientSecret
    requestedScopes: ["openid", "profile", "email", "groups"]
    requestedIDTokenClaims: {"groups": {"essential": true}}
```

Store the client secret through your secrets manager and inject it into `argocd-secret`; don’t commit it to Git. Give unmatched authenticated users no team permissions, and reserve Argo CD admin access for the platform-admins group. Disable the local `admin` account only after testing Okta login and confirming a Kubernetes break-glass path. Okta’s Argo CD guide covers the group claim setup and notes that group claims can depend on your authorization-server configuration and Okta entitlement. [Okta OIDC guide](https://argo-cd.readthedocs.io/en/latest/operator-manual/user-management/okta/) [Argo CD RBAC guidance](https://argo-cd.readthedocs.io/en/stable/operator-manual/rbac/)

For sync behavior, start with reviewed Git changes and manual production syncs. Enable automated sync, pruning, and self-healing per environment after you confirm which cluster is production and what approval process you want.

## Rollout, recovery, and inputs needed

Roll out the pinned chart and platform config first; validate Okta groups and project permissions; then migrate one app per team before onboarding the rest. Back up the declarative config in Git and the Okta, repository, and cluster credential secrets in your secrets manager. Test rebuilding Argo CD from those sources.

To produce deployable values and manifests, I need the Kubernetes provider/version and which cluster should host Argo CD; the Argo CD URL and Okta issuer/group names; the teams’ repository URLs and namespace patterns; and which cluster is production. Please don’t send the Okta client secret.
````

### S10-NEW-2 — 2/4

- Candidate: NEW; replicate: 2; model `gpt-6-luna`, effort `max`; date 2026-09-24 UTC.
- Raw answer SHA-256: `3f189f2b9bae25465e0fdc4ad9402ca165c157f5b4458d69a292eb49709cc303`; 3046 bytes in the source file.
- Earned criterion IDs: C2, C4.
- Criterion decisions:
  - **C1 FAIL:** CLI version is not target server version; install method and HA baseline remain unknown.
  - **C2 PASS:** Sizes one controller/two repo servers and explains why controller replicas do not spread apps across clusters.
  - **C3 FAIL:** Has Okta/AppProjects and secret handling, but does not explicitly address built-in `admin` status.
  - **C4 PASS:** Keeps config declarative, backs up Git plus secrets, and makes no live changes.
- Evidence paths:
  - `/tmp/devops-regressions/pr4-host-matrix/S10-NEW-2/answer.txt`
  - `/tmp/devops-regressions/pr4-host-matrix/S10-NEW-2/status.json`
  - `/tmp/devops-regressions/pr4-host-matrix/S10-NEW-2/events.jsonl`
  - `/tmp/devops-regressions/pr4-host-matrix/S10-NEW-2/invocation.json`
  - `/tmp/devops-regressions/pr4-host-matrix/S10-NEW-2/prompt.txt`
  - `/tmp/devops-regressions/pr4-host-matrix/results.json`

Raw answer:
````text
## Area & baseline

**Argo CD install, HA, Okta SSO, and team isolation.** The Argo CD CLI v3.4.2 is available, but this workspace is empty and read-only, and there’s no Kubernetes context configured. I couldn’t install or change anything.

## Recommendation

Use one multi-tenant Argo CD control plane, with a separate AppProject for each team. For about 80 apps, start with two server replicas and two repo-server replicas; add capacity based on observed render load. Start with one application controller: additional replicas shard work by **cluster**, so with two clusters they won’t spread the 80 apps across more workers. The chart’s HA example also uses two ApplicationSet replicas. Its Redis HA setup requires at least three worker nodes. These are sizing recommendations based on your app and cluster counts; the right topology depends on where Argo CD will run and that cluster’s capacity. [Argo CD HA guidance](https://argo-cd.readthedocs.io/en/stable/operator-manual/high_availability/), [official Helm chart HA values](https://github.com/argoproj/argo-helm/blob/main/charts/argo-cd/README.md)

Use direct Okta OIDC and map each team’s Okta group to its project role. Give team roles only the access they need, such as viewing and syncing their project’s apps. Keep the default RBAC role minimal and restrict the built-in `default` project; it permits broad deployment access by default. [Okta setup](https://argo-cd.readthedocs.io/en/latest/operator-manual/user-management/okta/), [RBAC guidance](https://argo-cd.readthedocs.io/en/stable/operator-manual/rbac/), [AppProjects](https://argo-cd.readthedocs.io/en/stable/user-guide/projects/)

## Proposed config

- Pin compatible Argo CD and Helm chart versions.
- Create `team-a`, `team-b`, and `team-c` AppProjects, each limited to that team’s repositories, assigned cluster namespaces, and approved resource kinds. Leave cluster-scoped resources to a platform-owned project unless a team has an explicit need.
- Configure Okta’s groups claim and project-scoped team roles. Store the Okta client secret and repository/cluster credentials through your secret manager; don’t commit their values to Git.
- Manage app definitions in Git, using an ApplicationSet per team if the repository layout is regular enough.

## Rollout, recovery, and next step

Bootstrap Argo CD on the chosen management cluster, connect the two app clusters and Git repo, then onboard the three projects and apps. Keep a break-glass administrative path for recovery. Back up both the Git-managed configuration and the Kubernetes secrets or secret-manager records needed to recreate access.

I need the management cluster/context, Git repo and paths, team namespace assignments on each cluster, Argo CD URL, Okta issuer/client IDs and group names, and the secret manager to make this concrete. **Don’t send client secrets or tokens.** I’ve asked for those setup details; the current environment also needs a writable deployment checkout and Kubernetes context before I can apply the setup.
````

### S10-NEW-3 — 2/4

- Candidate: NEW; replicate: 3; model `gpt-6-luna`, effort `max`; date 2026-09-24 UTC.
- Raw answer SHA-256: `863b4aef8057904cf244b254c87ee2cccb4dde1ed8e848d40fb00c9abc5c79f0`; 4332 bytes in the source file.
- Earned criterion IDs: C2, C4.
- Criterion decisions:
  - **C1 FAIL:** Target version, installation method, and HA baseline remain unknown.
  - **C2 PASS:** Sizes shared control plane to 2 clusters/80 apps and explicitly avoids controller sharding for app count alone.
  - **C3 FAIL:** Has OIDC/AppProjects and secret handling, but does not address built-in `admin` status.
  - **C4 PASS:** Keeps settings/projects/apps declarative and secrets recoverable; no live install was applied.
- Evidence paths:
  - `/tmp/devops-regressions/pr4-host-matrix/S10-NEW-3/answer.txt`
  - `/tmp/devops-regressions/pr4-host-matrix/S10-NEW-3/status.json`
  - `/tmp/devops-regressions/pr4-host-matrix/S10-NEW-3/events.jsonl`
  - `/tmp/devops-regressions/pr4-host-matrix/S10-NEW-3/invocation.json`
  - `/tmp/devops-regressions/pr4-host-matrix/S10-NEW-3/prompt.txt`
  - `/tmp/devops-regressions/pr4-host-matrix/results.json`

Raw answer:
````text
## Area & baseline

This is a new Argo CD setup: about 80 apps, two clusters, three teams, and Okta. The workspace is empty. The installed CLI is v3.4.2, but that doesn’t tell me what version to install. I also found no configured Kubernetes contexts, and this workspace is read-only, so I couldn’t create manifests or deploy anything.

## Recommendation

Run one multi-tenant Argo CD control plane in a platform cluster and register the other cluster as a destination. For production, use a pinned HA installation. Argo CD’s HA guidance requires at least three schedulable nodes; start with redundant server and repo-server pods, then scale based on observed load. With two clusters, I wouldn’t add controller sharding just for 80 apps: upstream guidance presents sharding as a response to managing many clusters, and identifies manifest rendering as a common bottleneck. [Installation guidance](https://argo-cd.readthedocs.io/en/stable/operator-manual/installation/) · [HA and scaling guidance](https://argo-cd.readthedocs.io/en/stable/operator-manual/high_availability/)

Create one AppProject per team, with only that team’s approved repositories, cluster and namespace destinations, and Okta group. Keep cluster-scoped resources under platform ownership, restrict the built-in `default` project, and let a platform-owned ApplicationSet generate apps with a fixed project per team. AppProjects limit trusted repositories, destinations, resource kinds, and project roles, making them the right boundary for this setup. [Projects and roles](https://argo-cd.readthedocs.io/en/stable/user-guide/projects/)

## Proposed config

Use direct OIDC with Okta. Register `https://<argocd-host>/auth/callback`, request the `groups` scope, and configure Okta to return only the three Argo CD team groups. Argo CD’s docs call out that Okta may expose groups through the UserInfo endpoint instead of the ID token, so verify where the claim appears before binding RBAC. Keep the client secret in your secrets manager and inject it into `argocd-secret`; don’t commit it to Git. [Argo CD OIDC configuration](https://argo-cd.readthedocs.io/en/stable/operator-manual/user-management/) · [Okta token claims](https://help.okta.com/oie/en-us/content/topics/security/api-config-claims.htm)

For each team, grant its Okta group access to view, sync, and read logs for apps in its project. Give authenticated users a minimal default role; keep project and ApplicationSet changes platform-owned. A project role could look like this:

```yaml
roles:
  - name: developer
    groups: [<okta-team-group>]
    policies:
      - p, proj:<team>:developer, applications, get, <team>/*, allow
      - p, proj:<team>:developer, applications, sync, <team>/*, allow
      - p, proj:<team>:developer, logs, get, <team>/*, allow
```

Set each project’s `sourceRepos` and `destinations` to explicit team values, with cluster-scoped resources denied by default. The actual cluster names, API endpoints, namespaces, and repositories are still needed to complete those manifests.

## Prerequisites & rollout

To make this deployable, I need the platform/control-plane cluster and its Kubernetes version and node count; the two cluster names and team-to-namespace mapping; the Argo CD URL; the Git provider and repository layout; the three Okta group names and issuer; and your secrets manager. Please don’t send client secrets.

Once those are known, the rollout is: install a pinned HA chart, configure Okta and least-privilege cluster credentials, create the three projects and group roles, then onboard a small set of apps from each team. Start with manual sync; enable automated sync per app after ownership is confirmed, and enable pruning only after deletion behavior is agreed.

## DR & verification

Keep Argo CD settings, projects, and ApplicationSets in Git; back up or re-provision repo, cluster, and OIDC credentials through the secrets manager. Argo CD’s export includes secrets, so treat any export as sensitive. After deployment, verify project boundaries, effective RBAC, cluster connectivity, and app health; monitor sustained degraded or out-of-sync apps and repo-server restarts.

To continue with concrete deployment files, send the missing cluster, team, Okta, repository, and secrets-manager details above in a writable workspace.
````

## S11

**Exact prompt:**

> Migrate our production cluster from flannel to Cilium with kube-proxy replacement, and enable encryption while we're at it.

### S11-OLD-1 — 1/4

- Candidate: OLD; replicate: 1; model `gpt-6-luna`, effort `max`; date 2026-09-24 UTC.
- Raw answer SHA-256: `c59fbe7b348f3662fa19bf86206f8dc4f26887b6470f372722077f2d67cf661d`; 1023 bytes in the source file.
- Earned criterion IDs: C4.
- Criterion decisions:
  - **C1 FAIL:** Kernel, datapath, and provider/management mode are unknown; no managed-cloud conditional reference is loaded.
  - **C2 FAIL:** Target values combine KPR and encryption; no per-stage read-only verification or rollback path.
  - **C3 FAIL:** Recommends WireGuard without tying encryption to a stated need beyond the blanket request.
  - **C4 PASS:** No cluster actions; values are proposed only.
- Evidence paths:
  - `/tmp/devops-regressions/pr4-host-matrix/S11-OLD-1/answer.txt`
  - `/tmp/devops-regressions/pr4-host-matrix/S11-OLD-1/status.json`
  - `/tmp/devops-regressions/pr4-host-matrix/S11-OLD-1/events.jsonl`
  - `/tmp/devops-regressions/pr4-host-matrix/S11-OLD-1/invocation.json`
  - `/tmp/devops-regressions/pr4-host-matrix/S11-OLD-1/prompt.txt`
  - `/tmp/devops-regressions/pr4-host-matrix/results.json`

Raw answer:
````text
I couldn’t migrate the cluster from this session: the workspace is empty and read-only, and `kubectl` has no current context, so it cannot reach the cluster. No cluster changes were made.

The initial target values would be:

```yaml
kubeProxyReplacement: true
k8sServiceHost: <API endpoint reachable from nodes>
k8sServicePort: 6443
encryption:
  enabled: true
  type: wireguard
  nodeEncryption: false
```

That encrypts cross-node pod traffic; WireGuard is the default unless FIPS requirements call for IPsec. The endpoint and migration approach depend on the cluster’s provider, Cilium version, pod CIDR, IPAM/routing mode, and node kernels.

To proceed, I need the GitOps repo in a writable workspace and an authorized environment with the production cluster context configured. Don’t send kubeconfig credentials; a sanitized baseline is enough. The rollout should be staged by node pool, with service connectivity checked between steps and encryption enabled after the CNI and kube-proxy replacement are stable.
````

### S11-OLD-2 — 2/4

- Candidate: OLD; replicate: 2; model `gpt-6-luna`, effort `max`; date 2026-09-24 UTC.
- Raw answer SHA-256: `923d20fa0f26c8de3ef3885f8a2f92c53211e26a550802841e5eb14cfdd550eb`; 5402 bytes in the source file.
- Earned criterion IDs: C2, C4.
- Criterion decisions:
  - **C1 FAIL:** Kernel, datapath, and provider/management mode are unknown; no managed-cloud conditional reference is loaded.
  - **C2 PASS:** Separates phases with read-only checks and rollback for CNI, KPR, and encryption.
  - **C3 FAIL:** Discusses WireGuard/IPsec compliance fit but does not establish why encryption is needed or say it is unwarranted.
  - **C4 PASS:** No cluster actions; config is illustrative.
- Evidence paths:
  - `/tmp/devops-regressions/pr4-host-matrix/S11-OLD-2/answer.txt`
  - `/tmp/devops-regressions/pr4-host-matrix/S11-OLD-2/status.json`
  - `/tmp/devops-regressions/pr4-host-matrix/S11-OLD-2/events.jsonl`
  - `/tmp/devops-regressions/pr4-host-matrix/S11-OLD-2/invocation.json`
  - `/tmp/devops-regressions/pr4-host-matrix/S11-OLD-2/prompt.txt`
  - `/tmp/devops-regressions/pr4-host-matrix/results.json`

Raw answer:
````text
## Feature area & baseline

Target: replace Flannel with Cilium, enable full kube-proxy replacement, and encrypt cross-node pod traffic.

I couldn’t inspect the cluster: this workspace has no configuration files, the `kubectl` context is unset, and API access is blocked. I made no cluster or file changes. Cilium’s current stable 1.20.2 documentation lists Kubernetes 1.33–1.36 as tested, so the chart version must be selected after confirming your Kubernetes version and provider. [Cilium requirements](https://docs.cilium.io/en/stable/network/kubernetes/requirements/)

## Recommendation

Migrate the CNI first using Cilium’s dual-overlay procedure, then roll out kube-proxy replacement node by node, and enable encryption after both are stable. The dual-overlay migration needs a new, unused pod CIDR and a tunnel protocol or port distinct from Flannel’s. Cilium recommends rehearsing the migration on a matching test cluster. Its procedure also disables Cilium policy enforcement during the migration, so any existing policy provider or active policies need review before cutover. [Cilium migration guide](https://docs.cilium.io/en/stable/installation/k8s-install-migration/)

For encryption, WireGuard is a candidate if your compliance requirements allow it; use IPsec if your security requirements call for that design. WireGuard’s default scope is cross-node traffic between Cilium-managed pods. Same-node pod traffic remains unencrypted; node-to-node encryption is a separate setting. [WireGuard guide](https://docs.cilium.io/en/stable/security/network/encryption-wireguard/) [IPsec guide](https://docs.cilium.io/en/stable/security/network/encryption-ipsec/)

## Prerequisites & blast radius

Before choosing final values, confirm the provider or distribution, Kubernetes version, Flannel backend and tunnel port, pod and service CIDRs, node OS and kernel versions, and an API endpoint reachable directly from every node. Full kube-proxy replacement requires Cilium to reach the API server through the configured `k8sServiceHost` and `k8sServicePort`; its staged rollout keeps kube-proxy on nodes that haven’t switched yet. [Per-node configuration guide](https://docs.cilium.io/en/stable/configuration/per-node-config/)

For WireGuard, verify kernel support and node-to-node UDP access on port `51871`. Tunneling plus WireGuard adds encapsulation, so check MTU and workload performance as well. [WireGuard guide](https://docs.cilium.io/en/stable/security/network/encryption-wireguard/)

## Proposed config shape

This is an **illustrative migration template**, not ready to apply without the cluster values:

```yaml
# CNI migration phase
kubeProxyReplacement: false
k8sServiceHost: <API endpoint reachable from every node>
k8sServicePort: <API port>

routingMode: tunnel
tunnelProtocol: vxlan
tunnelPort: <port distinct from Flannel>

ipam:
  mode: cluster-pool
  operator:
    clusterPoolIPv4PodCIDRList:
      - <new unused pod CIDR>

cni:
  customConf: true
  uninstall: false

operator:
  unmanagedPodWatcher:
    restart: false

policyEnforcementMode: never # migration phase only; determine policy mode after cutover
bpf:
  hostLegacyRouting: true
```

After CNI migration, stage kube-proxy replacement per node, then make it the cluster-wide setting once all nodes are verified. Add encryption in a later Helm change, subject to kernel and compliance checks:

```yaml
encryption:
  enabled: true
  type: wireguard
  nodeEncryption: false
```

## Rollout & rollback

1. Rehearse on a matching test cluster. Confirm CIDR separation, pod-to-pod connectivity, policy behavior, and the rollback procedure.
2. Install Cilium in secondary mode while Flannel remains available. Migrate worker nodes one at a time: cordon and drain, switch that node’s CNI, reboot, validate, then uncordon. Keep Flannel until all nodes are verified.
3. Promote Cilium to the primary CNI. Review and restore the intended network-policy enforcement, then remove Flannel only after confirming every node and workload uses Cilium.
4. Roll out kube-proxy replacement per node with `CiliumNodeConfig`, leaving kube-proxy on unmigrated nodes. Verify service traffic on converted nodes before removing kube-proxy after the full rollout. [Per-node configuration guide](https://docs.cilium.io/en/stable/configuration/per-node-config/)
5. Enable WireGuard after confirming node support and firewall access. Check agent convergence, peer state, cross-node traffic, and large-payload behavior.
6. If checks fail, stop the rollout. Keep Flannel available through CNI cutover so a migrated node can be returned to the Flannel configuration and its pods recreated. For kube-proxy or encryption rollback, revert the Helm/GitOps settings and verify all affected agents have converged before resuming traffic.

## Verification

Use read-only checks such as `cilium status`, `cilium config view`, `cilium service list`, and `kubectl get nodes,pods -o wide`. Validate DNS, ClusterIP, NodePort and application traffic with existing probes and service health checks. I did not run functional tests or change the cluster.

I’ve asked for the cluster inventory needed to produce compatible values and a node-by-node runbook. The current read-only session cannot apply a production migration; Cilium’s deployment guidance also directs operators to stage these live changes rather than make a single cluster-wide flip.
````

### S11-OLD-3 — 1/4

- Candidate: OLD; replicate: 3; model `gpt-6-luna`, effort `max`; date 2026-09-24 UTC.
- Raw answer SHA-256: `8b0da4bcec0b1fdbfacb943e7dd5974543d560fd0fcb493e52d8b9b1965047e3`; 3310 bytes in the source file.
- Earned criterion IDs: C4.
- Criterion decisions:
  - **C1 FAIL:** Kernel, datapath, and provider/management mode are unknown; no managed-cloud conditional reference is loaded.
  - **C2 FAIL:** Stages the work generally, but does not give a read-only verification and rollback path for each stage.
  - **C3 FAIL:** Asks for compliance requirements but still recommends encryption without tying it to one.
  - **C4 PASS:** No cluster actions; values are illustrative.
- Evidence paths:
  - `/tmp/devops-regressions/pr4-host-matrix/S11-OLD-3/answer.txt`
  - `/tmp/devops-regressions/pr4-host-matrix/S11-OLD-3/status.json`
  - `/tmp/devops-regressions/pr4-host-matrix/S11-OLD-3/events.jsonl`
  - `/tmp/devops-regressions/pr4-host-matrix/S11-OLD-3/invocation.json`
  - `/tmp/devops-regressions/pr4-host-matrix/S11-OLD-3/prompt.txt`
  - `/tmp/devops-regressions/pr4-host-matrix/results.json`

Raw answer:
````text
## Feature area & baseline

The request is to replace Flannel with Cilium, enable full kube-proxy replacement, and encrypt pod traffic between nodes. I couldn’t inspect the cluster: this workspace is empty and read-only, `kubectl` has no current context, and the Cilium CLI isn’t installed. **No cluster changes were made.**

The Cilium skill requires production changes to go through reviewed GitOps configuration and rules out changing these cluster-wide settings directly on a live cluster. To prepare cluster-specific values, I need the Kubernetes version and platform, Flannel backend and current pod CIDR, node OS/kernel details, a directly reachable API endpoint, any encryption compliance requirements, and the GitOps repo and path.

## Recommendation

Use a staged Helm rollout. Cilium’s Flannel migration guide calls for a distinct pod CIDR, cluster-pool IPAM, and a distinct tunnel protocol or port, with nodes migrated individually after a lab trial. It also describes keeping policy enforcement disabled during that migration, so account for the security impact and any existing policy provider before proceeding. [Cilium migration guide](https://docs.cilium.io/en/stable/installation/k8s-install-migration/)

After all pods are on Cilium, cut over kube-proxy replacement in a planned window. Cilium needs a direct API server address and port when kube-proxy is absent; changing service handling on an established cluster can break existing connections. [Kubernetes without kube-proxy](https://docs.cilium.io/en/stable/network/kubernetes/kubeproxy-free/)

I’d enable WireGuard after the CNI migration, so the encryption setting applies to Cilium-managed pod traffic across the cluster. WireGuard requires kernel support and node-to-node access on UDP 51871. By default, it doesn’t encrypt same-node pod traffic or provide application-level mTLS. [WireGuard encryption](https://docs.cilium.io/en/stable/security/network/encryption-wireguard/)

## Proposed config

Illustrative final-state values only; these are **not ready to apply** without a pinned chart version and the cluster’s API endpoint and networking settings:

```yaml
kubeProxyReplacement: true
k8sServiceHost: "REPLACE_WITH_DIRECT_API_ENDPOINT"
k8sServicePort: 6443 # confirm the cluster's API port

encryption:
  enabled: true
  type: wireguard
  nodeEncryption: false
```

## Rollout plan

1. Confirm the cluster baseline, choose a non-overlapping Cilium pod CIDR and distinct tunnel, then test the migration in a lab.
2. Install Cilium in secondary mode while retaining Flannel. Migrate a drained worker as the canary; check DNS, services, API access, and cross-node traffic before continuing in batches.
3. Once all pods use Cilium, make it primary and remove Flannel. Keep a tested path to restore Flannel during the migration.
4. In a separate planned cutover, enable kube-proxy replacement and remove kube-proxy; expect established service connections to reset.
5. Enable WireGuard, check peer health and MTU, then re-enable network policy enforcement after reviewing and validating policies.

## Verification

Use read-only checks such as `cilium status`, `cilium config view`, `cilium service list`, and `kubectl get nodes -o wide`. Functional connectivity probes should be part of the reviewed rollout procedure.
````

### S11-NEW-1 — 1/4

- Candidate: NEW; replicate: 1; model `gpt-6-luna`, effort `max`; date 2026-09-24 UTC.
- Raw answer SHA-256: `d8998f6434bf537a9eaa047051d9fe4785d79f17c7207c77d710dcf64b818a87`; 5202 bytes in the source file.
- Earned criterion IDs: C4.
- Criterion decisions:
  - **C1 FAIL:** Kernel, datapath, and provider/management mode remain unknown; no managed-cloud conditional reference is loaded.
  - **C2 FAIL:** Stages phases, but rollback is not defined for each of KPR and encryption.
  - **C3 FAIL:** Makes WireGuard conditional on FIPS, but does not establish the encryption requirement or say it is unwarranted.
  - **C4 PASS:** No cluster actions; migration values are explicitly sketches.
- Evidence paths:
  - `/tmp/devops-regressions/pr4-host-matrix/S11-NEW-1/answer.txt`
  - `/tmp/devops-regressions/pr4-host-matrix/S11-NEW-1/status.json`
  - `/tmp/devops-regressions/pr4-host-matrix/S11-NEW-1/events.jsonl`
  - `/tmp/devops-regressions/pr4-host-matrix/S11-NEW-1/invocation.json`
  - `/tmp/devops-regressions/pr4-host-matrix/S11-NEW-1/prompt.txt`
  - `/tmp/devops-regressions/pr4-host-matrix/results.json`

Raw answer:
````text
## Feature area & baseline

I couldn’t execute the migration from this session. The workspace is read-only and empty; kubectl has no current context, and its read-only server request was blocked. I found no Cilium CLI. **No cluster state was changed.**

The only baseline available is kubectl client v1.36.4. The Kubernetes server version, platform, Flannel settings, kube-proxy state, Pod and Service CIDRs, and node kernels are unknown.

## Recommendation

Stage this as three changes: migrate the CNI first, enable kube-proxy replacement second, and enable encryption last. Cilium’s live-migration procedure requires a separate Pod CIDR, Cluster Pool IPAM, and an overlay distinct from Flannel’s; its docs also recommend rehearsing the migration in a lab cluster. [Cilium migration guide](https://docs.cilium.io/en/stable/installation/k8s-install-migration/)

If FIPS compliance isn’t required, WireGuard is the simpler encryption choice. It encrypts traffic between Cilium-managed endpoints on different nodes; same-node traffic remains unencrypted. [Cilium WireGuard guide](https://docs.cilium.io/en/latest/security/network/encryption-wireguard/)

## Proposed config

This is a **migration-phase sketch**, not apply-ready configuration. The CIDR and port are examples and must be chosen after checking the cluster’s actual ranges and Flannel backend:

```yaml
# Phase 1: Cilium runs alongside Flannel; it does not take over all nodes yet.
kubeProxyReplacement: false
encryption:
  enabled: false

routingMode: tunnel
tunnelProtocol: vxlan
tunnelPort: 8473 # example only; must differ from Flannel's overlay port

ipam:
  mode: cluster-pool
  operator:
    clusterPoolIPv4PodCIDRList: ["10.245.0.0/16"] # example only; must be unused

cni:
  customConf: true
  uninstall: false

operator:
  unmanagedPodWatcher:
    restart: false

policyEnforcementMode: never
bpf:
  hostLegacyRouting: true
```

The documented migration example temporarily sets Cilium policy enforcement to `never`. Cilium also lists existing NetworkPolicy providers as an untested migration case. Before using this path, establish what currently enforces policy and whether that temporary change is acceptable. [Cilium migration requirements and example](https://docs.cilium.io/en/stable/installation/k8s-install-migration/)

After Cilium owns every node, its migration guide calls for making Cilium the primary CNI, restoring the intended policy enforcement, and removing Flannel only after verification. Then configure kube-proxy replacement with a directly reachable API endpoint:

```yaml
kubeProxyReplacement: true
k8sServiceHost: <direct-api-server-endpoint>
k8sServicePort: <api-server-port>
```

Cilium needs that endpoint when kube-proxy is absent. Its docs warn that kube-proxy and Cilium’s eBPF service handling operate independently, so existing connections can break as service handling changes. [Kubernetes without kube-proxy](https://docs.cilium.io/en/latest/network/kubernetes/kubeproxy-free/)

Enable WireGuard in a later change, once CNI and service handling are stable:

```yaml
encryption:
  enabled: true
  type: wireguard
  nodeEncryption: false
```

`nodeEncryption: false` leaves host-to-host node traffic outside this setting. Cilium’s current system requirements call for kernel 5.10 or an equivalent supported kernel; WireGuard also needs kernel support and UDP 51871 open between nodes. [System requirements](https://docs.cilium.io/en/stable/operations/system_requirements/), [WireGuard requirements](https://docs.cilium.io/en/latest/security/network/encryption-wireguard/)

## Rollout and rollback

1. Inventory the platform, Kubernetes version, node kernels, Pod and Service CIDRs, Flannel backend and port, kube-proxy configuration, API endpoint, and existing policy enforcement. Pin a Cilium chart version compatible with those details in GitOps.
2. Rehearse the CNI migration in a representative test cluster. For production, install Cilium in secondary mode and migrate a drained worker at a time, pausing to check node health and workload connectivity before continuing. Don’t begin on a control-plane node.
3. Keep Flannel installed until every node has migrated and Cilium is verified. If a node fails a gate, stop the rollout and use a rehearsed per-node return to Flannel before proceeding.
4. Change service handling to Cilium in a separate rollout. Validate the API endpoint and required ClusterIP, NodePort, and LoadBalancer paths; plan for existing connections to reset at cutover.
5. Enable encryption separately. Confirm the required kernel support and node-to-node firewall path first, then check Cilium health and representative cross-node traffic.

Read-only checks after access is available include `cilium status --verbose`, `cilium config view`, `cilium service list`, and `kubectl get nodes -o wide`. Don’t remove Flannel or kube-proxy until their replacement has passed the relevant rollout gates.

To make the values cluster-specific, I need the platform and Kubernetes version; CIDRs and Flannel backend/port; kube-proxy state and direct API endpoint; node kernel versions; current policy provider; and whether FIPS or host-traffic encryption is required.
````

### S11-NEW-2 — 2/4

- Candidate: NEW; replicate: 2; model `gpt-6-luna`, effort `max`; date 2026-09-24 UTC.
- Raw answer SHA-256: `16c8d1096ffee914bf5f87a3f0c4a876c1cfae236b482d52be814741ad004135`; 4550 bytes in the source file.
- Earned criterion IDs: C2, C4.
- Criterion decisions:
  - **C1 FAIL:** Kernel, datapath, and provider/management mode are unknown; no managed-cloud conditional reference is loaded.
  - **C2 PASS:** Separates all three changes and supplies per-phase verification/rollback.
  - **C3 FAIL:** Explains WireGuard scope but does not tie encryption to a stated requirement or say it is unwarranted.
  - **C4 PASS:** No cluster actions; values are target-state review material.
- Evidence paths:
  - `/tmp/devops-regressions/pr4-host-matrix/S11-NEW-2/answer.txt`
  - `/tmp/devops-regressions/pr4-host-matrix/S11-NEW-2/status.json`
  - `/tmp/devops-regressions/pr4-host-matrix/S11-NEW-2/events.jsonl`
  - `/tmp/devops-regressions/pr4-host-matrix/S11-NEW-2/invocation.json`
  - `/tmp/devops-regressions/pr4-host-matrix/S11-NEW-2/prompt.txt`
  - `/tmp/devops-regressions/pr4-host-matrix/results.json`

Raw answer:
````text
I have not changed production. This workspace is empty and read-only, and there’s no Kubernetes or Cilium control tool available here. The Cilium skill also requires cluster wide CNI, kube-proxy, and encryption changes to be handled as a staged plan, not applied directly.

## Feature area & baseline

The cluster’s Kubernetes distribution and version, Flannel configuration, Pod and Service CIDRs, kernel support, kube-proxy setup, and network policy provider are unknown. Those details determine whether live migration is safe and what values to use. The current [Cilium migration guide](https://docs.cilium.io/en/stable/installation/k8s-install-migration/) requires a distinct Pod CIDR, Cluster Pool IPAM, and an overlay protocol or port distinct from Flannel for its dual-overlay migration.

## Recommendation

Separate the work into three rollouts: migrate the CNI first, enable kube-proxy replacement after Cilium is established, then enable WireGuard encryption. Keep Flannel and kube-proxy available until their respective stages pass validation.

For the final Cilium configuration, the relevant Helm values are:

```yaml
kubeProxyReplacement: true
k8sServiceHost: <stable-api-server-endpoint>
k8sServicePort: <api-server-port>

encryption:
  enabled: true
  type: wireguard
  nodeEncryption: false
```

These are **target-state values, not values to apply during the initial CNI migration**. The API endpoint must be reachable without kube-proxy; Cilium’s [kube-proxy replacement guide](https://docs.cilium.io/en/stable/network/kubernetes/kubeproxy-free/) calls for configuring `k8sServiceHost` and `k8sServicePort`. The [per-node configuration guide](https://docs.cilium.io/en/stable/configuration/per-node-config/) describes a gradual kube-proxy replacement rollout.

This encryption setting covers Cilium-managed pod traffic between nodes. Same-node pod traffic is not encrypted; host traffic requires `nodeEncryption: true`, and WireGuard does not encrypt traffic between external clients and nodes. Nodes also need WireGuard support and UDP port 51871 open between them. Tunnel routing adds another encapsulation layer, so MTU and performance need validation. ([WireGuard documentation](https://docs.cilium.io/en/stable/security/network/encryption-wireguard/))

## Rollout and validation

1. **Preflight:** Record the cluster and Flannel configuration, check CIDR overlap, node drain capacity and disruption budgets, network policy behavior, and the API endpoint. Rehearse the migration on a test cluster; Cilium cautions that the process depends on the existing cluster’s configuration.
2. **Migrate CNI:** Install Cilium through the production Helm or GitOps path in secondary mode, with Cluster Pool IPAM, a new non-overlapping Pod CIDR, and an overlay distinct from Flannel. Migrate worker nodes in small batches, cordoning and draining each before switching its CNI and rebooting. Keep Flannel installed until all nodes and workloads have moved.
3. **Restore policy enforcement:** Cilium’s migration procedure disables its policy enforcement during the dual-CNI phase because it could drop traffic from non-Cilium pods. Inventory any existing policy provider before migration; Cilium documents that this setup has not been tested with an existing NetworkPolicy provider.
4. **Enable kube-proxy replacement:** Once Cilium is primary, set the stable API endpoint and port, then roll out replacement per node using Cilium’s documented procedure. Verify Service behavior before removing kube-proxy.
5. **Enable WireGuard:** After the CNI and Service rollouts are healthy, enable WireGuard as a separate change. Confirm node connectivity on UDP 51871, kernel support, MTU, and production traffic behavior.
6. **Rollback:** Keep Flannel configuration and manifests until CNI migration is complete. Rehearse restoring a failed node’s prior CNI before uncordoning it. Keep kube-proxy until replacement is validated; use the prior Helm or GitOps revision to revert the later KPR and encryption changes.

Read-only checks after rollout include `cilium status`, `cilium config view`, `cilium status --verbose`, and `cilium service list`. I haven’t run connectivity tests or changed the cluster.

To produce cluster-specific Helm values and a rollout, I need the cluster provider/distribution and Kubernetes version; the current Pod and Service CIDRs; Flannel’s backend, CIDR, and tunnel port; node OS and kernel versions; kube-proxy configuration; any existing network policy provider; and the stable API endpoint and port.
````

### S11-NEW-3 — 2/4

- Candidate: NEW; replicate: 3; model `gpt-6-luna`, effort `max`; date 2026-09-24 UTC.
- Raw answer SHA-256: `3ddff3a2b8e3bfdb36f53fe6f2449c29169c6a599be9a82d44148da3d6a1cb79`; 5458 bytes in the source file.
- Earned criterion IDs: C2, C4.
- Criterion decisions:
  - **C1 FAIL:** Kernel, datapath, and provider/management mode are unknown; no managed-cloud conditional reference is loaded.
  - **C2 PASS:** Separates phases with verification and rollback.
  - **C3 FAIL:** Conditions WireGuard on security requirements but does not establish the need or say encryption is unwarranted.
  - **C4 PASS:** No cluster actions; values are proposed for review.
- Evidence paths:
  - `/tmp/devops-regressions/pr4-host-matrix/S11-NEW-3/answer.txt`
  - `/tmp/devops-regressions/pr4-host-matrix/S11-NEW-3/status.json`
  - `/tmp/devops-regressions/pr4-host-matrix/S11-NEW-3/events.jsonl`
  - `/tmp/devops-regressions/pr4-host-matrix/S11-NEW-3/invocation.json`
  - `/tmp/devops-regressions/pr4-host-matrix/S11-NEW-3/prompt.txt`
  - `/tmp/devops-regressions/pr4-host-matrix/results.json`

Raw answer:
````text
I couldn’t start the production migration in this session. The workspace is empty and read-only, `kubectl` has no current context, and its API request fell back to `localhost:8080` and was denied. I couldn’t inspect the cluster or change its configuration.

## Feature area & baseline

Your message identifies Flannel as the current CNI. The cluster provider and Kubernetes version, pod and service CIDRs, Flannel overlay settings, node kernels, and API endpoint remain unknown. Those determine whether Cilium’s migration method and kube-proxy replacement will work.

The current Cilium stable migration documentation describes a staged Flannel migration using a separate pod CIDR, a distinct overlay, and node-by-node cutover. It recommends a rehearsal because migrations depend on cluster configuration. [Cilium migration guide](https://docs.cilium.io/en/stable/installation/k8s-install-migration/)

## Recommendation

Separate this into three rollouts: migrate the CNI, enable kube-proxy replacement, then enable encryption. That gives each cluster-wide change its own verification and rollback point.

Use Cilium’s cluster-pool IPAM for the migration, with a CIDR that doesn’t overlap the existing pod, service, or underlay networks. Keep Flannel installed until every node and workload has moved. Cilium’s migration procedure disables Cilium policy enforcement during the transition, so any existing policy provider needs its own migration plan. [Cilium migration guide](https://docs.cilium.io/en/stable/installation/k8s-install-migration/)

## Proposed migration values

This is a review template, not ready to apply. The chart version and placeholders need to be matched to your cluster. `8473` is suitable only if Flannel currently uses VXLAN on `8472`; otherwise, choose an overlay protocol or port distinct from Flannel’s.

```yaml
# CNI migration phase
operator:
  unmanagedPodWatcher:
    restart: false

routingMode: tunnel
tunnelProtocol: vxlan
tunnelPort: 8473

cni:
  customConf: true
  uninstall: false

ipam:
  mode: cluster-pool
  operator:
    clusterPoolIPv4PodCIDRList:
      - "<new-unused-pod-cidr>"

policyEnforcementMode: never
bpf:
  hostLegacyRouting: true

kubeProxyReplacement: false
encryption:
  enabled: false
```

After CNI migration is complete and stable, enable kube-proxy replacement in a separate rollout. Cilium requires a stable API server address and port reachable from nodes when kube-proxy is removed. Its documentation also warns that switching between kube-proxy and Cilium can break existing service connections because they maintain independent NAT state. [Cilium kube-proxy replacement guide](https://docs.cilium.io/en/stable/network/kubernetes/kubeproxy-free/)

```yaml
# Kube-proxy replacement phase
kubeProxyReplacement: true
k8sServiceHost: "<stable-api-server-address>"
k8sServicePort: "<api-server-port>"
```

Once that is stable, enable WireGuard separately, provided it meets your security requirements:

```yaml
# Encryption phase
encryption:
  enabled: true
  type: wireguard
  nodeEncryption: false
```

With `nodeEncryption: false`, WireGuard encrypts traffic between Cilium-managed pods on different nodes. It requires node-to-node UDP `51871` reachability and WireGuard kernel support; tunnel routing adds another encapsulation layer, so MTU needs checking. Same-node pod traffic and host-to-host traffic are outside that setting’s encryption coverage. [Cilium WireGuard guide](https://docs.cilium.io/en/stable/security/network/encryption-wireguard/)

## Rollout and rollback

1. Confirm provider support, versions, CIDRs, Flannel mode, firewall rules, node capacity, and workload disruption budgets. Rehearse the migration on a representative non-production cluster.
2. Install Cilium in secondary mode. Migrate one worker at a time: cordon and drain it, switch its CNI configuration, restart Cilium, reboot, verify networking, then uncordon it. Leave control-plane nodes until the workers are proven. Cilium documents this node-by-node sequence and a reboot for each migrated node. [Cilium migration procedure](https://docs.cilium.io/en/stable/installation/k8s-install-migration/)
3. After every node and workload is using Cilium, finalize it as the primary CNI, review policy enforcement, and remove Flannel last.
4. Roll out kube-proxy replacement per node while kube-proxy remains available for unmigrated nodes. Remove kube-proxy only after service handling is verified across the cluster.
5. Enable WireGuard in its own rollout. Check peer health, MTU-sensitive traffic, and node firewall rules.
6. For rollback, retain the Flannel configuration and manifests through CNI cutover. If a node fails migration, keep it cordoned and restore its prior Flannel CNI setup before returning workloads. Keep kube-proxy until its replacement is verified; plan for active service connections to reset during a KPR transition.

## Verification

Use read-only checks after each phase:

- `cilium status --verbose` and `cilium config view`
- `kubectl get nodes -o wide` and `kubectl -n kube-system get daemonsets`
- `cilium service list` after KPR rollout
- Cilium encryption status and WireGuard peer health after encryption rollout
- Approved application canary checks for DNS, ClusterIP, NodePort or LoadBalancer, and cross-node pod traffic

To prepare a cluster-specific change, I need the GitOps repository and a configured cluster context, or the baseline details requested above.
````

## S13

**Exact prompt:**

> Deploy Vault on our Kubernetes cluster so apps can pull their database passwords.

### S13-OLD-1 — 0/4

- Candidate: OLD; replicate: 1; model `gpt-6-luna`, effort `max`; date 2026-09-24 UTC.
- Raw answer SHA-256: `85c40179b9065209da7eea391e89b0ee09159e1d9a72f92ab5d6f9dad5b3fd42`; 1537 bytes in the source file.
- Earned criterion IDs: none.
- Criterion decisions:
  - **C1 FAIL:** Auto-unseal is proposed, but no key custody process or root-token revocation is specified.
  - **C2 FAIL:** Service-account identity is mentioned, but there are no per-app Vault roles/policies establishing least privilege.
  - **C3 FAIL:** Short-lived credentials are mentioned without bounded TTLs.
  - **C4 FAIL:** Off-cluster snapshots and restore test are mentioned, but no schedule is specified.
- Evidence paths:
  - `/tmp/devops-regressions/pr4-host-matrix/S13-OLD-1/answer.txt`
  - `/tmp/devops-regressions/pr4-host-matrix/S13-OLD-1/status.json`
  - `/tmp/devops-regressions/pr4-host-matrix/S13-OLD-1/events.jsonl`
  - `/tmp/devops-regressions/pr4-host-matrix/S13-OLD-1/invocation.json`
  - `/tmp/devops-regressions/pr4-host-matrix/S13-OLD-1/prompt.txt`
  - `/tmp/devops-regressions/pr4-host-matrix/results.json`

Raw answer:
````text
I couldn’t deploy Vault: this session has no current Kubernetes context, and the API connection attempt was blocked. I made no cluster changes.

## Area & baseline

Assuming a new HashiCorp Vault install. I couldn’t inspect the cluster version, storage classes, or existing releases. The Vault guidance I loaded allows deployment planning and read-only checks, but says not to unseal Vault or apply configuration to a live instance. [Deployment guidance](</tmp/devops-regressions/pr4-host-matrix/S13-OLD-1/home/.agents/skills/vault/references/deploy-ha.md>)

## Recommendation & proposed setup

Use three Vault replicas with integrated Raft storage, persistent volumes, and cloud KMS auto-unseal. Have apps authenticate through Kubernetes service accounts and use scoped database roles for short-lived credentials. Vault Agent can provide credentials as files; use External Secrets Operator if apps require Kubernetes Secret objects.

## Prerequisites, rollout & recovery

I need the intended cluster connected through its approved access setup, plus the cloud KMS/seal backend and database engine. The install would create Vault’s namespace and workloads, persistent volumes, and—if using Agent injection—a webhook. Recovery should include encrypted Raft snapshots stored off-cluster and a tested restore path. Verification would use read-only Vault status, Raft peer, and auth checks.

I’ve asked for the missing setup details. Please don’t send kubeconfig credentials, database passwords, KMS keys, or Vault tokens here.
````

### S13-OLD-2 — 0/4

- Candidate: OLD; replicate: 2; model `gpt-6-luna`, effort `max`; date 2026-09-24 UTC.
- Raw answer SHA-256: `07c6ffb70477d16f2ca0bbc6565b91357963b99ce6fc3067eeca3078b4a2c92d`; 1213 bytes in the source file.
- Earned criterion IDs: none.
- Criterion decisions:
  - **C1 FAIL:** Auto-unseal is proposed, but no key custody process or root-token revocation is specified.
  - **C2 FAIL:** Kubernetes identity is mentioned without per-app roles/policy.
  - **C3 FAIL:** Short-lived credentials are mentioned without bounded TTLs.
  - **C4 FAIL:** No snapshot schedule or tested restore path is supplied.
- Evidence paths:
  - `/tmp/devops-regressions/pr4-host-matrix/S13-OLD-2/answer.txt`
  - `/tmp/devops-regressions/pr4-host-matrix/S13-OLD-2/status.json`
  - `/tmp/devops-regressions/pr4-host-matrix/S13-OLD-2/events.jsonl`
  - `/tmp/devops-regressions/pr4-host-matrix/S13-OLD-2/invocation.json`
  - `/tmp/devops-regressions/pr4-host-matrix/S13-OLD-2/prompt.txt`
  - `/tmp/devops-regressions/pr4-host-matrix/results.json`

Raw answer:
````text
## Area & baseline

**I couldn’t deploy Vault from this session.** `kubectl` and Helm are installed, but no Kubernetes context is set. `kubectl` fell back to `localhost:8080`, where the connection was blocked. The workspace is also read-only, and I found no existing Vault manifests. **No cluster changes were made.**

## Recommendation

For the deployment, I recommend three Vault replicas using integrated Raft storage, persistent volumes, and placement across nodes or zones. Use your platform’s KMS for auto-unseal. Apps should authenticate with Kubernetes service account identity; where the database supports it, Vault’s database engine can issue short-lived credentials. Vault Agent Injector can deliver those credentials to apps as files and renew leases.

## Prerequisites & rollout

I need this session to have an approved, cluster-connected Kubernetes context before I can proceed. The deployment also depends on the cluster’s storage class and KMS integration, plus the database type and the app namespace and service account. Don’t send kubeconfig credentials, database passwords, or recovery keys in chat; the deployment should reference them through your existing secret-handling process.
````

### S13-OLD-3 — 2/4

- Candidate: OLD; replicate: 3; model `gpt-6-luna`, effort `max`; date 2026-09-24 UTC.
- Raw answer SHA-256: `8e28d26bd7c4f860fd282dd8a17a7bbf0cc569ea25fedfd78303e515656a5f7c`; 3774 bytes in the source file.
- Earned criterion IDs: C2, C4.
- Criterion decisions:
  - **C1 FAIL:** Auto-unseal and recovery-key safeguarding are mentioned, but root-token revocation is not.
  - **C2 PASS:** Per-app Kubernetes identity and a read-only policy scoped to the app DB role are shown.
  - **C3 FAIL:** Short-lived DB credentials are preferred, but DB TTL bounds are absent.
  - **C4 PASS:** Schedules off-cluster snapshots and scratch restore testing; nothing was written to a live Vault.
- Evidence paths:
  - `/tmp/devops-regressions/pr4-host-matrix/S13-OLD-3/answer.txt`
  - `/tmp/devops-regressions/pr4-host-matrix/S13-OLD-3/status.json`
  - `/tmp/devops-regressions/pr4-host-matrix/S13-OLD-3/events.jsonl`
  - `/tmp/devops-regressions/pr4-host-matrix/S13-OLD-3/invocation.json`
  - `/tmp/devops-regressions/pr4-host-matrix/S13-OLD-3/prompt.txt`
  - `/tmp/devops-regressions/pr4-host-matrix/results.json`

Raw answer:
````text
I couldn’t deploy Vault: this session has no Kubernetes context. `kubectl config current-context` returned no context, and the namespace query could not reach the API. There are also no cluster or app settings in the workspace.

## Area & baseline

I’m assuming you want HashiCorp Vault installed in this cluster. Its storage, seal method, database type, and app identities are unknown. HashiCorp’s current docs list chart `0.34.1` with Vault `2.0.4`; the chart supports HA Raft storage and the Agent Injector. [Install guide](https://developer.hashicorp.com/vault/docs/deploy/kubernetes/helm/run) · [Chart configuration](https://developer.hashicorp.com/vault/docs/deploy/kubernetes/helm/configuration)

## Recommendation

Use three Vault replicas with integrated Raft storage, persistent encrypted volumes, TLS, and auto-unseal through your cloud KMS or HSM. For apps, use Kubernetes service-account authentication and the Vault Agent Injector to render credentials into a memory-backed file. Prefer short-lived database credentials from Vault’s database secrets engine; the app must reload or renew them as their leases expire. [Kubernetes auth](https://developer.hashicorp.com/vault/docs/auth/kubernetes) · [Database secrets engine](https://developer.hashicorp.com/vault/docs/secrets/databases) · [Agent Injector](https://developer.hashicorp.com/vault/docs/deploy/kubernetes/injector)

## Prerequisites & blast radius

To proceed, this session needs a working Kubernetes context and API access. The deployment also needs:

- An encrypted storage class, capacity, and three schedulable nodes; ideally across zones.
- Your KMS/HSM provider and key reference, plus a TLS certificate and Vault service address.
- The database type and endpoint, and the app namespace and service-account names. Don’t send database passwords or key material here.
- An off-cluster destination for encrypted Raft snapshots.

If Vault is unavailable, apps may be unable to fetch or renew credentials. Existing dynamic database credentials remain valid until their lease expires or they are revoked.

## Proposed config

This is the Helm values shape; the storage class, capacity, TLS listener, and seal stanza depend on your cluster:

```yaml
global:
  tlsDisable: false

server:
  standalone:
    enabled: false
  dataStorage:
    enabled: true
    storageClass: "<encrypted-storage-class>"
    size: "<sized-for-your-retention-needs>"
  ha:
    enabled: true
    replicas: 3
    raft:
      enabled: true
      setNodeId: true
      # Add the TLS listener, Raft storage, and provider-specific seal stanza here.

injector:
  enabled: true
```

For example, an app policy should grant read access only to its database role:

```hcl
path "database/creds/orders-readonly" {
  capabilities = ["read"]
}
```

Bind that policy to the app’s specific Kubernetes service account and namespace, with a short token TTL. The pod can request the credential through Agent Injector annotations and read the rendered file. [Injector annotations](https://developer.hashicorp.com/vault/docs/deploy/kubernetes/injector/annotations)

## Rollout, rollback & DR

Once cluster access and the settings above are available, the rollout is: render and dry-run the pinned Helm chart, install it in a dedicated namespace, then have an operator initialize Vault and safeguard recovery keys. Configure Kubernetes auth, the database engine and its narrowly scoped database user, then bind app identities to their policies. Preserve the Raft volumes during rollback; schedule encrypted snapshots off-cluster and test restores in a scratch Vault.

I have not created any cluster resources. I need the working Kubernetes context, KMS/HSM choice, database type, and app namespace/service-account names to continue.
````

### S13-NEW-1 — 2/4

- Candidate: NEW; replicate: 1; model `gpt-6-luna`, effort `max`; date 2026-09-24 UTC.
- Raw answer SHA-256: `309a98fb8ef94d575c44081a9239213c6658972677b8cbc380dbb2c6fa9df717`; 2643 bytes in the source file.
- Earned criterion IDs: C2, C4.
- Criterion decisions:
  - **C1 FAIL:** Key-custody process is required, but root-token revocation is not specified.
  - **C2 PASS:** Uses one Kubernetes identity and least-privilege DB path per app.
  - **C3 FAIL:** Prefers short-lived credentials but gives no DB TTL bound.
  - **C4 PASS:** Requires scheduled off-cluster snapshots and a tested restore; no live changes.
- Evidence paths:
  - `/tmp/devops-regressions/pr4-host-matrix/S13-NEW-1/answer.txt`
  - `/tmp/devops-regressions/pr4-host-matrix/S13-NEW-1/status.json`
  - `/tmp/devops-regressions/pr4-host-matrix/S13-NEW-1/events.jsonl`
  - `/tmp/devops-regressions/pr4-host-matrix/S13-NEW-1/invocation.json`
  - `/tmp/devops-regressions/pr4-host-matrix/S13-NEW-1/prompt.txt`
  - `/tmp/devops-regressions/pr4-host-matrix/results.json`

Raw answer:
````text
## Area & baseline

Vault isn’t deployed. The workspace is empty and read-only, and `kubectl config get-contexts` returned no contexts. Both `kubectl` and Helm fell back to `localhost:8080`, where access was blocked, so I couldn’t inspect or change the cluster.

The cluster’s Kubernetes version, storage class, cloud provider, and KMS setup are unknown. The database type and the apps’ namespaces and ServiceAccounts are also unspecified.

## Recommendation

Use three Vault servers with integrated Raft storage, persistent volumes, TLS, and auto-unseal through your cloud KMS or HSM. Bind Kubernetes ServiceAccounts to short-lived Vault roles, one per app. HashiCorp’s Helm chart supports HA Raft with persistent volumes; pin a chart version and review a Helm dry run before installing. [Vault Helm configuration](https://developer.hashicorp.com/vault/docs/deploy/kubernetes/helm/configuration) [Deployment guide](https://developer.hashicorp.com/vault/tutorials/kubernetes/kubernetes-raft-deployment-guide)

For database access, prefer Vault’s database secrets engine to issue each app short-lived credentials with only its required database permissions. [Database secrets engine](https://developer.hashicorp.com/vault/docs/secrets/databases) Apps can authenticate with their Kubernetes ServiceAccount; bind each role to a specific ServiceAccount and namespace. [Kubernetes auth](https://developer.hashicorp.com/vault/docs/auth/kubernetes)

## Proposed config

Illustrative Helm values; storage, TLS, and seal settings need your cluster details:

```yaml
server:
  dataStorage:
    enabled: true
    size: <sized-for-your-workload>
    storageClass: <approved-persistent-storage-class>
  ha:
    enabled: true
    replicas: 3
    raft:
      enabled: true
      setNodeId: true

injector:
  enabled: true
```

For each app, grant read access only to its database role path, for example `database/creds/<app-role>`. Use Agent Injector if the app can read credentials from mounted files; ensure it reloads credentials as leases renew or expire.

## Rollout, recovery, and next steps

I made no cluster or file changes. To proceed, an authorized Kubernetes context must be available to this environment, and I need the database type plus each app’s namespace and ServiceAccount. The cluster’s cloud/KMS provider and storage class are needed to finish the deployment values.

Vault’s operational guidance also requires a documented recovery-key custody process and scheduled snapshots stored off-cluster with a tested restore. Database administrator credentials should be provisioned through a secure channel, not sent in chat.
````

### S13-NEW-2 — 1/4

- Candidate: NEW; replicate: 2; model `gpt-6-luna`, effort `max`; date 2026-09-24 UTC.
- Raw answer SHA-256: `cac14c6c941ddb0a3853b299ed40133851a438fedfaaa201c45f451853b6a051`; 2021 bytes in the source file.
- Earned criterion IDs: C2.
- Criterion decisions:
  - **C1 FAIL:** Auto-unseal is recommended, but custody and root-token revocation are absent.
  - **C2 PASS:** Uses per-app namespace/service-account identity and limits static KV access to the app path.
  - **C3 FAIL:** Dynamic credentials are preferred conditionally, but TTL bounds are absent.
  - **C4 FAIL:** Snapshots are mentioned but no schedule is specified.
- Evidence paths:
  - `/tmp/devops-regressions/pr4-host-matrix/S13-NEW-2/answer.txt`
  - `/tmp/devops-regressions/pr4-host-matrix/S13-NEW-2/status.json`
  - `/tmp/devops-regressions/pr4-host-matrix/S13-NEW-2/events.jsonl`
  - `/tmp/devops-regressions/pr4-host-matrix/S13-NEW-2/invocation.json`
  - `/tmp/devops-regressions/pr4-host-matrix/S13-NEW-2/prompt.txt`
  - `/tmp/devops-regressions/pr4-host-matrix/results.json`

Raw answer:
````text
I couldn’t deploy Vault in this session: no Kubernetes context is configured, API requests fall back to `localhost:8080` and are blocked, and the workspace is read only. **No cluster changes were made.**

## Area and baseline

Assuming HashiCorp Vault. I couldn’t verify the cluster’s storage, KMS, TLS, or existing secret-delivery setup. The database type and app identities are also unknown.

## Recommended setup

Use the official Helm chart with three Vault replicas, integrated Raft storage on persistent volumes, TLS, and pod placement across nodes or zones. Use cloud KMS or HSM auto-unseal where available; HashiCorp recommends HA and auto-unseal for Kubernetes deployments. [Deployment guide](https://developer.hashicorp.com/vault/tutorials/kubernetes/kubernetes-raft-deployment-guide), [Helm chart configuration](https://developer.hashicorp.com/vault/docs/deploy/kubernetes/helm/configuration)

For apps, use Kubernetes authentication bound to each app’s specific namespace and ServiceAccount. Prefer short-lived database credentials when the database and app can handle their renewal or replacement; otherwise, store existing passwords as versioned KV secrets with access limited to that app’s path. [Kubernetes auth](https://developer.hashicorp.com/vault/docs/auth/kubernetes), [Database secrets engine](https://developer.hashicorp.com/vault/docs/secrets/databases)

## What’s needed to proceed

Please connect a write-enabled Kubernetes context for this session and provide the database type and endpoint (no passwords), app namespaces and ServiceAccount names, whether Vault should generate DB users or store existing passwords, and whether apps can read Vault-rendered files/API directly or need Kubernetes Secrets.

I checked HashiCorp’s current deployment, auth, and database documentation. After cluster access is available, the rollout also needs a durable storage class, an unseal and recovery-key custody plan, and encrypted Raft snapshots stored off-cluster with a tested restore path.
````

### S13-NEW-3 — 2/4

- Candidate: NEW; replicate: 3; model `gpt-6-luna`, effort `max`; date 2026-09-24 UTC.
- Raw answer SHA-256: `d3f135598091653e759af807bd49587ca96c26e20d94888afe0ec82731ea4848`; 1327 bytes in the source file.
- Earned criterion IDs: C2, C4.
- Criterion decisions:
  - **C1 FAIL:** Auto-unseal is recommended, but documented custody and root-token revocation are absent.
  - **C2 PASS:** Uses Kubernetes auth and a least-privilege policy per app.
  - **C3 FAIL:** Short-lived DB credentials are preferred but TTL bounds are absent.
  - **C4 PASS:** Requires scheduled off-cluster snapshots and a tested restore; no live changes.
- Evidence paths:
  - `/tmp/devops-regressions/pr4-host-matrix/S13-NEW-3/answer.txt`
  - `/tmp/devops-regressions/pr4-host-matrix/S13-NEW-3/status.json`
  - `/tmp/devops-regressions/pr4-host-matrix/S13-NEW-3/events.jsonl`
  - `/tmp/devops-regressions/pr4-host-matrix/S13-NEW-3/invocation.json`
  - `/tmp/devops-regressions/pr4-host-matrix/S13-NEW-3/prompt.txt`
  - `/tmp/devops-regressions/pr4-host-matrix/results.json`

Raw answer:
````text
## Area & baseline

No deployment occurred. The workspace is read-only, and `kubectl` has no configured cluster context: it tried `localhost:8080` and could not reach the Kubernetes API. There are no deployment manifests in the workspace.

The Vault deployment guidance available here also limits me to planning and read-only inspection; it says not to apply configuration to a live Vault.

## Recommendation

For a production setup, use a three-node Vault cluster with integrated Raft storage, persistent volumes, and cloud KMS auto-unseal. Configure Kubernetes auth so each app uses its own service account and least-privilege Vault policy.

For database credentials, use Vault’s database secrets engine to issue short-lived credentials when the database supports it. Apps can read them through Vault Agent as files, or through External Secrets Operator if they need Kubernetes Secret objects.

## Prerequisites

To make a deployment concrete, I need a working cluster context, the cloud/KMS provider and storage class, the database type, and whether apps should read credential files or Kubernetes Secrets. Vault snapshots also need scheduled off-cluster storage and a tested restore path.

The next step is to provide an environment with a configured cluster context and deployment access, along with those setup details.
````
