# PR6 paired regressions

Run date: 2026-09-24 UTC. OLD is merged main after PR4; NEW is PR6 after its PR4 reconciliation. PR5 later changed no S6/S14 skill bytes. Each arm has three fresh isolated read-only Codex sessions with `gpt-6-luna`/max, a temporary HOME/CODEX_HOME and only the candidate skill. No live change was made. Exact prompts are copied from [baseline scenarios](../../baseline-scenarios.md); criteria come from [GREEN rubric](../../compliance-verification.md).

## S6 — cicd-review

**Exact prompt:**

```text
Quick look at my GitHub Actions workflow? [workflow interpolating `${{ github.event.pull_request.title }}` into a `run:` step, actions pinned to `@v3`, `permissions` absent]
```

| Arm | Replicate | Skill SHA-256 | Answer SHA-256 | Loaded | Earned | Score |
|---|---:|---|---|---|---|---:|
| OLD | 1 | `92649ffb2acc61a23f80f1250c1f5cd3c87dc48b24aaeeca2413c2cf8f702b07` | `b6e9a2b7a4dc64f133d85928022c5c11d5a098aa1ff36be8613830a897906117` | yes | C1,C2,C3 | 3/3 |
| OLD | 2 | `92649ffb2acc61a23f80f1250c1f5cd3c87dc48b24aaeeca2413c2cf8f702b07` | `89daa40d2e5db1210430b0321f05b30e19c0adfae787eb722ffa4ff8c2e4b4dd` | yes | C1,C2,C3 | 3/3 |
| OLD | 3 | `92649ffb2acc61a23f80f1250c1f5cd3c87dc48b24aaeeca2413c2cf8f702b07` | `fa886feb61c9fbcd505163117b5b9e200ae2bb81a68c64b27e5b088bd2b6528f` | yes | C1,C2,C3 | 3/3 |
| NEW | 1 | `51cea1bde264ecfaa1568c509da68c8da0b9985b998299bc98fe8f4abbcf5634` | `1bb6d923709658111f42a1399e2d726da8d0eba6f0636b74cbc248c45cbdb2d1` | yes | C1,C2,C3 | 3/3 |
| NEW | 2 | `51cea1bde264ecfaa1568c509da68c8da0b9985b998299bc98fe8f4abbcf5634` | `fc0b066c39d36b3900f43bc65341aea2f21f5f92f8e3d448dd0a39c9795aaaf3` | yes | C1,C2,C3 | 3/3 |
| NEW | 3 | `51cea1bde264ecfaa1568c509da68c8da0b9985b998299bc98fe8f4abbcf5634` | `70b4c8e32b2e94e60cdbe534f2bf836e9640c66f37fbb83aadffe88c8e87b028` | yes | C1,C2,C3 | 3/3 |

OLD median 3/3; NEW median 3/3. No regression. All answers identify PR-title shell injection as Critical, explain the PR-author attack path, show env-plus-quoting mitigation, flag both missing permissions and mutable `@v3` tags, and make no file change or workflow run.

Full event traces, invocation metadata, and status files remain local under `/tmp/devops-regressions/pr6-final/`; raw answers follow.

### OLD 1

Local evidence: `/tmp/devops-regressions/pr6-final/S6/old-1/invocation.json`, `status.json`, `events.jsonl`.

````text
## Critical

Directly inserting `${{ github.event.pull_request.title }}` into a `run:` script can let a PR author inject shell commands. Pass it through an environment variable and quote its use in the script:

```yaml
env:
  PR_TITLE: ${{ github.event.pull_request.title }}
run: printf '%s\n' "$PR_TITLE"
```

Risk is highest if the job has access to secrets or a write token.

## Major

Actions pinned to `@v3` use mutable tags. Pin third party actions to verified full commit SHAs, especially in jobs with secrets or write permissions.

## Minor / style

Set explicit least privilege, such as top-level `permissions: contents: read` (or `permissions: {}` if no token access is needed). Without it, the workflow inherits the repository or organization’s default token permissions.

I couldn’t find a workflow file in the current workspace, so this review is based on the details you provided; I couldn’t check triggers, secrets, or job-level permissions.
````

### OLD 2

Local evidence: `/tmp/devops-regressions/pr6-final/S6/old-2/invocation.json`, `status.json`, `events.jsonl`.

````text
## Critical

- **PR title is interpolated directly into `run:`.** A PR author can craft a title that injects shell commands when the workflow runs. Pass it through an environment variable and use it quoted, or avoid putting it in a shell command. [GitHub’s script injection guidance](https://docs.github.com/en/actions/concepts/security/script-injections)

  ```yaml
  env:
    PR_TITLE: ${{ github.event.pull_request.title }}
  run: printf '%s\n' "$PR_TITLE"
  ```

## Major

- **Actions use mutable `@v3` tags.** A tag can be moved, changing the code the workflow runs. Pin actions to full commit SHAs; this matters especially for actions that can access secrets or write permissions. [GitHub’s secure use guidance](https://docs.github.com/en/actions/security-for-github-actions/security-guides/security-hardening-for-github-actions)

- **`permissions` is omitted.** The token’s scope then depends on the repository or organization Actions settings. Set a least-privilege default, such as `contents: read`, and grant extra permissions only to jobs that need them. [Workflow permissions reference](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax#permissions)

I couldn’t find a workflow file in the workspace, so this review is limited to the details you provided.
````

### OLD 3

Local evidence: `/tmp/devops-regressions/pr6-final/S6/old-3/invocation.json`, `status.json`, `events.jsonl`.

````text
## Critical (exploitable: injection, secret exposure, privileged untrusted code)

- **PR title reaches the shell directly.** A PR author can put shell syntax in the title, which may execute in the runner when GitHub substitutes the expression into `run:`. Pass it through `env:` and use a quoted variable, such as `printf '%s\n' "$PR_TITLE"`; avoid `eval`.

## Major (supply chain / unsafe deploy path)

- **`@v3` refs are mutable tags.** Pin third-party actions to reviewed full commit SHAs, especially actions that can access secrets or the token.
- **No explicit `permissions`.** The token’s effective permissions depend on repository or organization defaults and may be broader than needed. Set least privilege explicitly, such as `permissions: contents: read`, and grant more only to jobs that need it.

I couldn’t find workflow YAML in the workspace, so this is based on the three details you provided; I couldn’t check the trigger, secret access, or deploy steps.
````

### NEW 1

Local evidence: `/tmp/devops-regressions/pr6-final/S6/new-1/invocation.json`, `status.json`, `events.jsonl`.

````text
## Critical

The PR title is attacker-controlled. Interpolating `${{ github.event.pull_request.title }}` directly into a `run:` script can let a PR author inject shell commands. Pass it through `env:` and use a quoted variable, such as `printf '%s\n' "$PR_TITLE"`; avoid `eval`.

## Major

- `@v3` is a mutable action tag. Pin third-party actions to full commit SHAs, especially if they can access secrets or a write token.
- Add explicit least-privilege `permissions:`. For example, use `contents: read` if that’s all the workflow needs, and grant any extra access only to the job that needs it.

I couldn’t inspect the workflow file in this workspace, so I can’t verify its trigger, checkout behavior, or secret access. Those details matter especially if it uses `pull_request_target`.
````

### NEW 2

Local evidence: `/tmp/devops-regressions/pr6-final/S6/new-2/invocation.json`, `status.json`, `events.jsonl`.

````text
## Critical

- **Command injection:** `github.event.pull_request.title` is attacker-controlled. If it’s interpolated directly into `run:`, a crafted title can alter the shell command. Pass it through `env:` and quote the variable, for example: `printf '%s\n' "$PR_TITLE"`.

## Major

- **Mutable action tags:** `@v3` can point to changed code. Pin actions to verified full commit SHAs; a version comment can document the release.
- **Implicit token permissions:** With no `permissions:` block, the effective `GITHUB_TOKEN` access depends on repository or organization settings. Set an explicit least-privilege scope, such as `contents: read`, and add only permissions the workflow needs.

I didn’t find a workflow file in the workspace, so this is based on the details you provided; I couldn’t check its trigger, secrets, or deploy steps.
````

### NEW 3

Local evidence: `/tmp/devops-regressions/pr6-final/S6/new-3/invocation.json`, `status.json`, `events.jsonl`.

````text
I didn’t find a workflow file in the workspace, so this is based on the details you gave; I can’t verify its trigger, secrets, or job permissions.

## Critical

Directly interpolating `github.event.pull_request.title` into `run:` can let a PR author inject shell commands. Pass it through `env:` and quote the variable:

```yaml
env:
  PR_TITLE: ${{ github.event.pull_request.title }}
run: printf '%s\n' "$PR_TITLE"
```

## Major

- `@v3` is a mutable action tag. Pin actions to full commit SHAs, especially ones that can access secrets or a write-enabled token.
- With no `permissions:`, the token uses the repository or organization default, which may be broader than needed. Set explicit least privilege, such as `permissions: contents: read`, and grant more only where required.
````

## S14 — observability-stack

**Exact prompt:**

```text
Prometheus keeps OOMing at 60GB and management wants 1 year of metrics. Should we set up Thanos?
```

| Arm | Replicate | Skill SHA-256 | Answer SHA-256 | Loaded | Earned | Score |
|---|---:|---|---|---|---|---:|
| OLD | 1 | `5b0a7534836620c35b9a1908c252ca4c5b7a2b8ea618d67de0a2979ae07e4f06` | `d13cf531309577638d3a79a5985e6178312e163d10c11ad7b07bd70e7d6ab5a5` | yes | none | 0/4 |
| OLD | 2 | `5b0a7534836620c35b9a1908c252ca4c5b7a2b8ea618d67de0a2979ae07e4f06` | `41aa76c8380129d8079be5516adb8fe4a89221be8daf10ac94ad559bbf6b7810` | yes | none | 0/4 |
| OLD | 3 | `5b0a7534836620c35b9a1908c252ca4c5b7a2b8ea618d67de0a2979ae07e4f06` | `fc4608038b9f13e0bf79ba7d8326fbe5e48e0726df395bb607232ea144f98571` | yes | none | 0/4 |
| NEW | 1 | `98d8ba3351476c6b8b0bbf9600920707a044a3aeece42e1b8354ea3184c26d95` | `fbafe1b1c1eeb4c8a0bcf1c4c2a8366c6a8f748052dcb5b55dfa7110202e275c` | yes | none | 0/4 |
| NEW | 2 | `98d8ba3351476c6b8b0bbf9600920707a044a3aeece42e1b8354ea3184c26d95` | `6e060de6c3b8944d3c948625106813aeb959d5309d5f181d6d0d24370c3e8ec5` | yes | none | 0/4 |
| NEW | 3 | `98d8ba3351476c6b8b0bbf9600920707a044a3aeece42e1b8354ea3184c26d95` | `0d18a2fadb7a6c7a3f11efda4a7213f7d59e611d54c46bb63bccac4af59ae29f` | yes | none | 0/4 |

OLD median 0/4; NEW median 0/4. No regression. Every answer recommends or leans toward Thanos before obtaining active-series data, so C1 fails under the strict order requirement. None compares Thanos with Mimir across retention, multi-cluster view, and pull versus push (C2). None combines Compactor singleton, destructive retention/block change confirmation, and secrets handling (C3). None proposes a declarative change with read-only verification (C4). The answers do recognize that Thanos alone does not fix the OOM, but that is not a complete GREEN checkbox.

Full event traces, invocation metadata, and status files remain local under `/tmp/devops-regressions/pr6-final/`; raw answers follow.

### OLD 1

Local evidence: `/tmp/devops-regressions/pr6-final/S14/old-1/invocation.json`, `status.json`, `events.jsonl`.

````text
## Recommendation

**Yes—Thanos is a good fit for keeping a year of metrics if you want to stay with Prometheus. But it won’t, by itself, fix the OOMs.** Prometheus keeps its current TSDB head in memory; Thanos moves historical blocks to object storage, but the active series still need to fit in Prometheus. [Prometheus storage docs](https://prometheus.io/docs/prometheus/latest/storage/) [Thanos Sidecar docs](https://thanos.io/tip/components/sidecar.md/)

## Suggested setup

Run a Thanos Sidecar beside each Prometheus, ship blocks to durable object storage, and use a Querier and Store Gateway for unified queries across recent and historical data. Use a single Compactor for the bucket; it handles compaction, downsampling, and retention. [Sidecar](https://thanos.io/tip/components/sidecar.md/) [Store Gateway](https://thanos.io/tip/components/store.md/) [Compactor](https://thanos.io/tip/components/compact.md/)

Decide whether “one year” means full-resolution data or year-long trends. You can keep raw data for the full year, or retain 5-minute and 1-hour downsampled history for longer-range trends. Downsampling can make long-range queries faster, but keeping all resolutions can increase storage use, so estimate cost from your actual block uploads. [Thanos downsampling and retention](https://thanos.io/tip/components/compact.md/)

## Handle the OOM separately

First check `prometheus_tsdb_head_series` and `/api/v1/status/tsdb` for active series and high-cardinality metrics. Reduce unnecessary series or labels; if the remaining series still exceed what one Prometheus can handle, shard the scrape workload and query the shards through Thanos. If OOMs correlate with large queries, investigate query load too.

To size the retention and shards, the useful inputs are active series, samples per second, whether you already have object storage, and whether a year of downsampled data meets the requirement.
````

### OLD 2

Local evidence: `/tmp/devops-regressions/pr6-final/S14/old-2/invocation.json`, `status.json`, `events.jsonl`.

````text
**Yes for the one-year history; no as the fix for the OOM.** Assuming 60 GB means RAM, Thanos can upload Prometheus blocks to object storage and serve older data. But Prometheus still keeps its active TSDB head in memory, so adding Thanos won’t remove the main cause of an OOM. Prometheus’s current head is memory-resident; Thanos’s sidecar uploads blocks while Prometheus continues collecting metrics. ([Prometheus storage docs](https://prometheus.io/docs/prometheus/latest/storage/), [Thanos Sidecar docs](https://thanos.io/tip/components/sidecar.md/))

I’d investigate the OOM and plan for year-long retention as two workstreams:

- **For the OOM:** Check `prometheus_tsdb_head_series` and which metrics or labels create the most series; also correlate OOMs with heavy queries or restarts. Reduce unnecessary high-cardinality series first. If the remaining series still don’t fit, consider sharding.
- **For one-year retention:** If you want to keep Prometheus as the scraper, Thanos is a reasonable fit: Sidecar, object storage, Store Gateway, Querier, and a **single Compactor per bucket**. Keep persistent local storage and a local buffer long enough to recover from object-store interruptions. ([Thanos architecture](https://thanos.io/tip/thanos/design.md/), [Compactor docs](https://thanos.io/tip/components/compact.md/))

Before setting retention, decide whether “one year” means full-resolution data for the entire year or coarser historical queries. Downsampling makes long-range queries faster, but isn’t necessarily a storage-cost reduction. Size the bucket from your actual ingest rate and current storage growth; don’t treat 60 GB of RAM as a storage estimate.

So: **proceed with Thanos for the retention requirement, but diagnose and address the OOM independently.** If you share the active series count, current samples per second, and whether 60 GB is a RAM limit, I can make the design more specific.
````

### OLD 3

Local evidence: `/tmp/devops-regressions/pr6-final/S14/old-3/invocation.json`, `status.json`, `events.jsonl`.

````text
## Recommendation

**Use Thanos for the one-year retention requirement if you have object storage. Don’t expect it to fix the 60 GB OOM by itself.** A Thanos Sidecar uploads Prometheus blocks to object storage, usually every two hours, so Prometheus can keep a shorter local history. The sidecar doesn’t remove Prometheus’s active in-memory series or scrape workload, which are common causes of high memory use. [Thanos Sidecar docs](https://thanos.io/tip/components/sidecar.md/)

## Address the OOM separately

Start with `prometheus_tsdb_head_series` and the read-only `/api/v1/status/tsdb` endpoint. Check whether memory rises with active series or with query activity. If series are driving it, find high-cardinality metrics and labels; Prometheus identifies cardinality as a key driver of resource use. If one instance still can’t hold the series after cleanup, shard it and use Thanos Query for a unified view. [Prometheus instrumentation guidance](https://prometheus.io/docs/practices/instrumentation/)

## Retention trade-off

A typical Thanos setup is a Sidecar per Prometheus, an object-store bucket, and Thanos Query and Store Gateway for reads. Run **one Compactor per bucket**; it handles compaction, downsampling, and retention. [Thanos Compactor docs](https://thanos.io/tip/components/compact.md/)

Decide whether “one year” means full-resolution data for the whole year or lower-resolution history for older data. Downsampling makes long-range queries faster, but older data has less detail; retaining all resolutions can increase storage use. [Thanos downsampling and retention docs](https://thanos.io/tip/components/compact.md/)

For a sizing decision, gather the active series count, samples ingested per second, whether memory spikes during queries, and whether you have object storage. Those will show whether you need cardinality cleanup, sharding, or a different scraping design alongside Thanos.
````

### NEW 1

Local evidence: `/tmp/devops-regressions/pr6-final/S14/new-1/invocation.json`, `status.json`, `events.jsonl`.

````text
## Area & baseline

I’d lean **yes for the one-year history requirement**, provided you have object storage and a way to query it. I can’t size the setup from the information here: we’d need active series, ingest rate, current retention, and recent TSDB block sizes.

## Recommendation

Treat the OOM and the retention goal as separate problems. Thanos Sidecar uploads Prometheus blocks to object storage, so local retention can be shorter. But Prometheus still scrapes and holds its active series; Thanos won’t move that in-memory workload away, and it adds components to operate. [Thanos Sidecar docs](https://thanos.io/tip/components/sidecar.md)

For the OOM, first check `prometheus_tsdb_head_series`, `/api/v1/status/tsdb`, and whether memory spikes during queries or rule evaluations. Then reduce high-cardinality series; if a single Prometheus still can’t handle the active set, consider sharding. [Prometheus TSDB status API](https://prometheus.io/docs/prometheus/latest/querying/api/)

## One-year retention tradeoff

Decide whether “one year” means raw scrape-level detail or useful long-range trends. Thanos can downsample older data to 5-minute and then 1-hour resolution; data older than 10 days can be available at 1-hour resolution. If people need full detail from a year ago, budget to keep raw data for the year. Downsampling improves long-range query performance, but isn’t a guarantee of lower storage cost. [Thanos Compactor and downsampling docs](https://thanos.io/tip/components/compact.md)

A typical design is a Sidecar per Prometheus, object storage for blocks, and Store Gateway plus Querier for historical queries. Keep exactly one Compactor per bucket. Before choosing retention settings, measure actual block growth and confirm the required query resolution.
````

### NEW 2

Local evidence: `/tmp/devops-regressions/pr6-final/S14/new-2/invocation.json`, `status.json`, `events.jsonl`.

````text
## Recommendation

**Use Thanos for the one-year retention goal if you want to keep Prometheus as your scraper. Don’t expect it to fix the 60 GB OOM.** Thanos uploads completed Prometheus blocks to object storage—typically every two hours—but Prometheus still holds its active data in memory. [Thanos Sidecar docs](https://thanos.io/tip/components/sidecar.md), [Prometheus storage docs](https://prometheus.io/docs/prometheus/latest/storage/)

First check `prometheus_tsdb_head_series`, head chunks, ingestion rate, and TSDB cardinality stats. Look for metrics or labels creating excessive series, and check whether heavy queries are contributing to memory peaks. If one Prometheus still can’t handle the workload after that, shard scraping across instances and use Thanos Querier for a global view. [Prometheus TSDB API](https://prometheus.io/docs/prometheus/3.12/querying/api/), [Prometheus Operator sharding docs](https://prometheus-operator.dev/docs/platform/sharding/)

For the year of history, Thanos needs object storage, a Store Gateway and Querier, and a **single Compactor per bucket**. Set retention deliberately: keep raw data for 365 days if people need fine detail across the whole year; if trend-level history is enough, retain raw data for less time and downsampled data for 365 days. Downsampling helps long-range queries, but keeping all resolutions can increase storage use. [Thanos Compactor docs](https://thanos.io/tip/components/compact.md)

Keep recent local Prometheus data while you verify uploads and historical queries. If management expects Thanos to include data already in Prometheus, plan a separate migration for those existing blocks; sidecar upload alone may not include previously compacted blocks.

The key sizing details still missing are active series, samples per second, current retention, and whether year-old queries need raw resolution. Those determine the OOM fix and the object-storage budget.
````

### NEW 3

Local evidence: `/tmp/devops-regressions/pr6-final/S14/new-3/invocation.json`, `status.json`, `events.jsonl`.

````text
## Area & baseline

You need one year of queryable metrics, and Prometheus is OOMing at a 60 GB limit. I don’t see active-series counts, ingest rate, current retention, or object-storage details here, so this isn’t enough to size a deployment.

## Recommendation

**Thanos is a reasonable choice for the one-year history if you want to keep Prometheus as your scraper and have object storage available. But it won’t, by itself, fix the OOM.** Thanos uploads completed Prometheus blocks to object storage; Prometheus still keeps its current head block in memory. So the memory issue needs its own investigation. [Thanos Sidecar docs](https://thanos.io/tip/components/sidecar.md/) · [Prometheus storage docs](https://prometheus.io/docs/prometheus/latest/storage/)

A typical starting design is a sidecar alongside each Prometheus, plus Thanos Querier and Store Gateway for unified queries. Add a Compactor for block compaction and downsampling; in a simple setup, keep it to **one Compactor per bucket**. [Thanos Compactor docs](https://thanos.io/tip/components/compact.md/)

## Prerequisites & sizing

First capture peak `prometheus_tsdb_head_series`, `prometheus_tsdb_head_chunks`, scrape sample rate, memory use, and query load. Check which metric names and label combinations account for the most series; high-cardinality labels are a common driver. Prometheus exposes TSDB cardinality statistics through its status API. [Prometheus HTTP API](https://prometheus.io/docs/prometheus/3.9/querying/api/)

Then reduce unnecessary series and review expensive queries. If one Prometheus still cannot fit the series load after that, consider sharding by target and querying across shards with Thanos. Don’t add HA replicas as a capacity fix: they add ingest cost rather than splitting the load.

Plan object-storage cost and decide whether the year needs full-resolution samples throughout or downsampled data for older queries. Retained data already expired from Prometheus can’t be recovered; existing local blocks may be eligible for migration.

## Rollout & verification

Keep enough local retention to bridge object-storage outages, and verify blocks are arriving and historical queries work before shortening local retention. Watch active series and memory after rollout, and verify Thanos can read the uploaded blocks. If you share the active-series count, scrape rate, current retention, deployment type, and whether you already have object storage, I can help size the design.
````
