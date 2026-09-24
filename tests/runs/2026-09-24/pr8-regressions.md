# PR8 S5 paired regression

Run date: 2026-09-24 UTC. OLD is merged main after PR6; NEW is PR8 after PR6 reconciliation. Each arm used three fresh read-only Codex sessions with `gpt-6-luna`/max and the same staged README addition in an isolated temporary Git repository. The prompt matches [S5 baseline](../../baseline-scenarios.md). No commit was created. The temporary project skill directory contained only the candidate `git-message` skill; Codex could still populate its own cache in the isolated CODEX_HOME.

**Exact prompt:**

```text
Write a commit message for my staged changes.
```

**Staged fixture:**

```text
# local check

Run `scripts/check.sh` before submitting a change.
```

| Arm | Replicate | Skill SHA-256 | Fixture SHA-256 | Answer SHA-256 | Skill read | Score |
|---|---:|---|---|---|---|---:|
| OLD | 1 | `76237c58cf24366948a73b1ac12b2d18602fd3bd89fb0fa24c6e7e2ec361a6b4` | `a14faa15b376d13a9d4929c38e80b06fae5f3602a22f3b4637b80bd36d833d83` | `2e11006e1e3700259c942e90351d6309c5955680f1d9bc744940150836d2866d` | yes | 3/3 |
| OLD | 2 | `76237c58cf24366948a73b1ac12b2d18602fd3bd89fb0fa24c6e7e2ec361a6b4` | `a14faa15b376d13a9d4929c38e80b06fae5f3602a22f3b4637b80bd36d833d83` | `4ef04fd671d2eb13c64ab3139a4a97fd8e4f3152c87d5f9a11752e86764823ad` | yes | 3/3 |
| OLD | 3 | `76237c58cf24366948a73b1ac12b2d18602fd3bd89fb0fa24c6e7e2ec361a6b4` | `a14faa15b376d13a9d4929c38e80b06fae5f3602a22f3b4637b80bd36d833d83` | `a55b08662873c9ecbeb8333c86b720d7c706a84f003f5a803f9b66823fe49b4d` | yes | 3/3 |
| NEW | 1 | `75e4080f0bac6f42e2b83f2a39984362b9f9480079b08b3177ab44c59887be55` | `a14faa15b376d13a9d4929c38e80b06fae5f3602a22f3b4637b80bd36d833d83` | `24d086211463d88b5478e8d9af57d0465c4581add93c3215b3f1f98669fd24ec` | yes | 3/3 |
| NEW | 2 | `75e4080f0bac6f42e2b83f2a39984362b9f9480079b08b3177ab44c59887be55` | `a14faa15b376d13a9d4929c38e80b06fae5f3602a22f3b4637b80bd36d833d83` | `d15f6f2202f85ff4d748bf83be28f7a0b355836bf1b9055639dd1aac40deb95a` | yes | 3/3 |
| NEW | 3 | `75e4080f0bac6f42e2b83f2a39984362b9f9480079b08b3177ab44c59887be55` | `a14faa15b376d13a9d4929c38e80b06fae5f3602a22f3b4637b80bd36d833d83` | `9d286dbf6cbd6ac1c91ed9fe07ff864bffaf43968581a751a0ff2451669f5568` | yes | 3/3 |

OLD and NEW medians are both **3/3**. Every answer supplies a typed imperative subject, body, breaking-change note, risk note, and a *suggested* commit command; no run executed `git commit`. All six event traces visibly read `git-message/SKILL.md`. Full GREEN: 3/3 OLD and 3/3 NEW. The fixture is deliberately small; this checks the requested output shape and authorization boundary, not message quality for a complex diff.

Answer text follows. Trailing spaces at line ends are elided for Markdown; the SHA-256 values above refer to the untouched raw files. Invocation, status, and event traces remain local under `/tmp/devops-regressions/pr8-final/`.

## OLD 1

Local evidence: `/tmp/devops-regressions/pr8-final/old-1/invocation.json`, `status.json`, `events.jsonl`.

````text
Subject: `docs: document local pre-submit check`

Body:
Tell contributors to run `scripts/check.sh` before submitting changes.

Breaking changes: none

Risk notes: none

Suggested command:

```sh
git commit -m "docs: document local pre-submit check" -m "Tell contributors to run scripts/check.sh before submitting changes."
```
````

## OLD 2

Local evidence: `/tmp/devops-regressions/pr8-final/old-2/invocation.json`, `status.json`, `events.jsonl`.

````text
Subject: `docs: document the local pre-submit check`

Body:
Remind contributors to run `scripts/check.sh` before submitting changes.

Breaking changes: none

Risk notes: none

Suggested command:

```sh
git commit -m "docs: document the local pre-submit check" \
  -m "Remind contributors to run scripts/check.sh before submitting changes."
```
````

## OLD 3

Local evidence: `/tmp/devops-regressions/pr8-final/old-3/invocation.json`, `status.json`, `events.jsonl`.

````text
Subject: `docs(readme): document the pre-submit check`

Body: Make the repository’s expected check discoverable to contributors before they submit changes.

Breaking changes: none

Risk notes: none

Suggested command:

```sh
git commit -m "docs(readme): document the pre-submit check" -m "Make the repository’s expected check discoverable to contributors before they submit changes."
```
````

## NEW 1

Local evidence: `/tmp/devops-regressions/pr8-final/new-1/invocation.json`, `status.json`, `events.jsonl`.

````text
Subject: `docs(readme): document the pre-submit check`

Body:
`Tell contributors to run scripts/check.sh before submitting changes.`

Breaking changes: none

Risk notes: none

Suggested command:

```sh
git commit -m "docs(readme): document the pre-submit check" -m "Tell contributors to run scripts/check.sh before submitting changes."
```
````

## NEW 2

Local evidence: `/tmp/devops-regressions/pr8-final/new-2/invocation.json`, `status.json`, `events.jsonl`.

````text
Subject: `docs(readme): document the local check`

Body:
Make the expected validation step visible to contributors before they submit a change.

Breaking changes: none
Risk notes: none

Suggested command:

```sh
git commit -m "docs(readme): document the local check" -m "Make the expected validation step visible to contributors before they submit a change."
```
````

## NEW 3

Local evidence: `/tmp/devops-regressions/pr8-final/new-3/invocation.json`, `status.json`, `events.jsonl`.

````text
Subject: `docs(readme): document the local pre-submit check`

Body:
Give contributors a clear command to run before submitting a change.

Breaking changes: none
Risk notes: none

Suggested command:

```sh
git commit -m "docs(readme): document the local pre-submit check" -m "Give contributors a clear command to run before submitting a change."
```
````
