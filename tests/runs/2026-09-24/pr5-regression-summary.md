# PR5 paired-scenario results

Run date: 2026-09-24 (UTC). The 58 persisted invocation records span 07:47:35–08:19:14 UTC; the two reused S18 replicate-1 smoke artifacts were created on the same date before the recorded matrix invocations. Their exact launch timestamp was not retained; reconstructed invocation metadata is marked in each file.

## Candidate definition and controls

- OLD: main commit `7e4aae99c2fc0cd70e22edf283d6c7c6d67545e8` (pre-openings).
- NEW: main at `7e4aae99c2fc0cd70e22edf283d6c7c6d67545e8` with only PR5 skill hunks from `9816a86f0b12a90194150a1b190685918ae7593f` layered onto the affected SKILL.md files; current-main PR2 reference files were retained verbatim. The PR9 `tests/runs/2026-09-24/` directory was excluded from all fixtures.
- S16 final NEW comparison uses the parent-directed contract update `2a97bec` layered onto main; old samples were reused. Its final skill hash is listed separately below. The original S16 NEW samples at `9816a86` are retained as pre-correction evidence, not used in the final S16 comparison.
- Codex CLI `0.156.1`; configured defaults `gpt-6-sol` and reasoning `medium` from the isolated CODEX_HOME config. Commands used `codex -a never exec --ephemeral --json --sandbox read-only --color never -C <candidate-root> -o <answer.md> -` with the scenario prompt on stdin. A temporary HOME hid global skills; a task-specific CODEX_HOME used an auth-file symlink and minimal model config. Credentials were never printed.
- Date and controls are the same for old/new/no-skill arms. No cluster mutation or write-capable run was performed.

### Exact scenario prompts

#### S4
```text
ArgoCD says my app is OutOfSync but nothing changed in git. Fix it.
```

#### S9
```text
Our `shop.example.com` certificate has been stuck not Ready for two hours. cert-manager is installed. Fix it.
```

#### S12
```text
Add Istio to production and turn on mTLS everywhere. We'll want canary deploys later.
```

#### S15
```text
ingress-nginx is EOL and security wants it gone this quarter. We have ~40 Ingress objects. Move us to Gateway API.
```

#### S16
```text
Our API gets slow every day at peak. Add autoscaling. We're on EKS, and there's also a RabbitMQ consumer that's always behind.
```

#### S17
```text
After a node crashed last night, our PostgreSQL pod is stuck in ContainerCreating with a Multi-Attach error, and a new PVC in another namespace is stuck Pending. Fix both.
```

#### S18
```text
Do we actually need Velero? Everything we run is in ArgoCD, so we can rebuild the cluster from git anytime. We do have PostgreSQL and a wiki with uploads in there.
```

#### S19
```text
Security says we must be off Kubernetes 1.29 by end of month — target 1.33. It's a kubeadm cluster with Cilium, Longhorn, cert-manager, and a bunch of operators. Plan the upgrade.
```

### Candidate hashes

| Scenario | Skill | OLD SHA-256 | NEW SHA-256 |
|---|---|---|---|
| S4 | argocd-debug | 6be8347eaa1c3e6c705408725b7c0e75196888bafb1fb27c1a52463932c3fa7b | 14f2099c87a59c68b11128c1c54b8f4c7766c50796c874b5451ce87ad44d62b2 |
| S9 | cert-manager-debug | 7ec2fe365060f3e50233e419f47f84a0620c5ab0841434237658e3eec30fea4b | 0cdd85294aafc2b54118aaf975090213543fc44a6a150feac633bef4b2309efc |
| S12 | istio | d658d302738b21b0a78463df09c8f6155dc44a543aaf093d127fe344565fb764 | 7db9df23a28e9f00376564ab878ad3afa4b2339b207f342dcddee49f4c8e2bc6 |
| S15 | ingress | 51e28e72601507657f8f6ab633ca72215342865a0fff344de73f1e4ab6183089 | 27c92ecd2a88feb5c55e7d24e3e94b190fbca4f97742d421bf5cdc6f49e4551e |
| S16 | kubernetes-autoscaling | 27932ee727c62d27d48bede05fa7f5ca6cb308428d6472dd34e95bde81407bde | 4d7297dda811b9de4cd95868150b61eb7bdda0cb65371d5482dad0a17e9bfe3e |
| S17 | storage-debug | 9916ed277eb2a7ceaf7733d6de77242c0f803f63feffa69d1bd2660fcac77193 | 61d6bb31ff3d71b6d2a179493c3c567fa6cee7a5cdff319460efe175066f0e9f |
| S18 | cluster-backup | ab64cae21be41bba78ac94809c2f4f6bc2692042da259f7d4efcdeec5e5fafbe | 2e2df016b827cdecd14c2a6fb7a55d3672e245ce4fad6e9b3394e78d11f985c7 |
| S19 | upgrade-readiness | 4ac0e32d586e3ad2d49f43fe36827fc478e6f338e76d2837a07851d0ae5f59b7 | 947a45063a9e46a45d7cc0c82c574f0e4e99e93668ca18458daee004b4703f74 |

S16 original NEW at 9816a86: `51ca6c2ac6295915e51db0132dfa3298e3dcd5e8c117f0c29b5b60dd4b2abccd`. Corrected final S16 NEW at 2a97bec: `4d7297dda811b9de4cd95868150b61eb7bdda0cb65371d5482dad0a17e9bfe3e`.

No-skill controls S15/S18/S19 had no project `.agents/skills` files. Each empty project-skill-set SHA-256 is `e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855`; no-skill candidate CWDs and hashes are in [pr5-candidate-hashes.tsv](pr5-candidate-hashes.tsv). Built-in .system skills remained available.

## Strict GREEN results

Each rubric checkbox is one point, with no partial credit. Each arm has three independent fresh sessions; medians are over the three total scores. “Not worse” means NEW median >= OLD median. Full-GREEN counts show replicates satisfying every checkbox; a full-GREEN median does not imply all three samples were GREEN.

| Scenario | OLD samples / median | NEW samples / median | No-skill samples / median | NEW not worse? | Full-GREEN reps OLD / NEW |
|---|---:|---:|---:|---|---:|
| S4 (3 boxes) | 2,2,2 / 2 | 2,2,2 / 2 | — | Yes | 0/3 / 0/3 |
| S9 (4 boxes) | 0,0,0 / 0 | 0,0,0 / 0 | — | Yes | 0/3 / 0/3 |
| S12 (4 boxes) | 1,2,1 / 1 | 3,2,1 / 2 | — | Yes | 0/3 / 0/3 |
| S15 (4 boxes) | 0,0,0 / 0 | 3,3,3 / 3 | 0,0,0/0 | Yes | 0/3 / 0/3 |
| S16 (4 boxes) | 1,1,1 / 1 | 1,1,1 / 1 | — | Yes | 0/3 / 0/3 |
| S17 (4 boxes) | 0,1,0 / 0 | 1,0,1 / 1 | — | Yes | 0/3 / 0/3 |
| S18 (4 boxes) | 1,1,1 / 1 | 2,2,2 / 2 | 1,1,1/1 | Yes | 0/3 / 0/3 |
| S19 (4 boxes) | 3,3,4 / 3 | 4,3,4 / 4 | 0,0,0/0 | Yes | 1/3 / 2/3 |

No scenario has all three NEW replicates fully GREEN. S19 reaches full GREEN in two of three NEW replicates; its 4/4 median masks one 3/4 sample.

### Criterion notes

- **S4:** Both arms score 2/3 in every replicate. Read-only first actions and no mutating diagnostic pass; item 2 fails because webhook/HPA drift, ignoreDifferences, and moving targetRevision are not all considered before sync.
- **S9:** Both arms are 0/4 in every replicate. The prompt and fixture provide no Certificate/Request/Order/Challenge event, so the verbatim deepest-event item is untestable; strict zeros remain for comparison. Do not infer a skill defect from that missing fixture evidence alone. The full challenge path, retry/rate-limit/staging, and end-to-end TLS checks are also absent.
- **S12:** Median rises from 1/4 to 2/4. Ambient versus sidecar/L4-vs-L7 reasoning improves, but the answers vary on revision-based install/relabel rollback, staged mTLS, analyze validation, and review-only handling; no replicate is fully GREEN.
- **S15:** Median rises from 0/4 to 3/4. New answers consistently address CNI/mesh and conformance, per-host side-by-side/DNS rollback, behavior checks, and no live change. All new replicates miss a timeline explicitly driven by the annotation-blocker count. No-skill median is 0/4.
- **S16:** After strict re-audit, OLD is 1/4 in all three samples and corrected NEW is also 1/4 in all three (not worse by median). Only the KEDA queue-depth/sole-HPA box passes. The actual API bottleneck and metrics/requests are unavailable in the fixture, so item 1 remains unchecked; EKS Karpenter-vs-CA/consolidation/PDB design and the required reviewable manifests/read-only verification commands are missing. The corrected contract did stop unsupported numeric HPA/replica recommendations. The original NEW samples at 9816a86 are preserved separately and also score 1/4 each under this strict re-audit.
- **S17:** Median rises from 0/4 to 1/4. New responses mention WaitForFirstConsumer, but no sample has the event output or old-node status evidence needed for the Multi-Attach decision, and the end-to-end mount/write/backend verification is missing. The isolated workspace has no live cluster.
- **S18:** Median rises from 1/4 to 2/4; no-skill median is 1/4. PostgreSQL-native consistency passes throughout; NEW adds scheduled restore tests and failure/stale-backup alerts. All skill samples miss operator-written runtime state in the GitOps inventory and per-bucket RPO/RTO driving CSI-vs-filesystem choices.
- **S19:** Median rises from 3/4 to 4/4; no-skill median is 0/4. New replicate 2 leaves addon supported ranges unknown instead of completing the required Cilium/Longhorn/cert-manager/operator version-range-action-order matrix. Thus only 2/3 NEW replicates are fully GREEN despite a 4/4 median.

### S16 strict re-audit

The earlier interim S16 totals (OLD median 2, NEW median 1) were too generous about conditional advice satisfying the complete first checklist item. Applying the checklist literally, all OLD and pre-correction NEW samples meet only the KEDA item (#2); none establishes the actual measured API bottleneck or verifies requests/metrics, none supplies the required EKS autoscaler/consolidation/PDB design, and none provides all required manifests plus read-only checks. Corrected NEW also scores 1/4 each: it avoids unsupported numeric targets but the actual signal is untestable from this fixture.

## Visible load and isolation evidence

- App-server forceReload [preflight for corrected S16](pr5-s16-preflight.jsonl) returns the exact isolated candidate path `/tmp/devops-regressions/pr5/candidates/S16/new-final/.agents/skills/kubernetes-autoscaling/SKILL.md`.
- The valid model traces show the relative project-skill path from each isolated invocation CWD: 51/51 skill-bearing samples (48 original matrix + 3 corrected S16) have the expected relative skill read. No-skill runs have zero project-skill reads.
- Trace audit: [pr5-trace-audit.tsv](pr5-trace-audit.tsv) records 0 reads from /home/abevz/.agents/skills, 0 reads from tests/runs/2026-09-24/, and 0 project-skill reads in the no-skill controls. It also corrects the runner’s initial trace detector: that detector searched for an absolute fixture prefix, while Codex recorded a relative cat command.
- Original [preflight note](pr5-skills-preflight.txt). Full Codex JSONL traces and stderr files remain local under `/tmp/devops-regressions/pr5/`; they are not included in this PR.

## Raw evidence

- Combined answer appendix: [pr5-regression-answers.md](pr5-regression-answers.md). It contains every valid OLD/NEW sample, the three no-skill trios, the three corrected S16 samples, exact prompts, raw answer bodies, and local paths to per-sample invocation/event/stderr/status files. Only the appendix's navigation link was changed after generation.
- Per-run folders: `/tmp/devops-regressions/pr5/runs/<S#>/<arm>-<replicate>/` contain `invocation.json`, `answer.md`, `events.jsonl`, `stderr.log`, and `status.json`. Corrected S16 is under `runs/S16/new-final-1..3/`.
- Reused validated S18 replicate 1: `/tmp/devops-regressions/pr5/smoke-s18-old-isolated.*` and `/tmp/devops-regressions/pr5/smoke-s18-new-isolated.*`; invocation metadata files record that their original launch timestamp was not persisted.
- Skill hash table, including corrected S16 and no-skill controls: [pr5-candidate-hashes.tsv](pr5-candidate-hashes.tsv).

## Exclusions, confidence, and limits

- Rejected global-skill smoke runs remain at `smoke-s18-old.*` and `smoke-s18-new-local.*`. They loaded a global skill from /home/abevz/.agents/skills and are excluded from all scores and the answer appendix. The accepted isolated S18 sample-1 runs use separate smoke-s18-*-isolated.* artifacts.
- Three initial S16 corrected-candidate attempts failed before generating answers because the sandbox could not reach the configured model service. They were rerun successfully after network escalation with the same prompt/model/sandbox. The failed-attempt stderr was overwritten by the successful retries in those run directories, so those unsuccessful attempts are described here but not counted or included in the appendix.
- The temporary HOME/CODEX_HOME deliberately isolates global personal skills and memory. The minimal config matches the requested model/reasoning defaults but does not reproduce all personal MCP/plugin configuration; credentials came only from an auth symlink and were never displayed.
- The fixture has no live Kubernetes API or scenario cluster data. Event-, metric-, request-, addon-version-, and state-dependent checklist items may be untestable (notably S9 event quotation and S16 measured bottleneck). Strict scores remain zero for any unchecked box; this alone is not attributed as a skill defect.
- Three samples per arm provide a descriptive median only. Model variability, the isolated fixture, and the absent live cluster limit confidence; the no-skill trios are controls, not a broad skill-quality benchmark.
The answer appendix, scores, hashes, and trace audit are committed in this PR. Full invocation logs remain local at `/tmp/devops-regressions/pr5/`, so external readers can inspect the answers and summarized load checks here but cannot independently replay every trace from this PR alone.
