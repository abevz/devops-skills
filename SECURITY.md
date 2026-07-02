# Security

This repository holds Agent Skills — instructions that get loaded into an AI coding agent's
context and can influence what commands it suggests or runs. Treat that as a real attack surface,
not just documentation.

## Third-party skills are untrusted input

Any skill you didn't write yourself — including ones you download to inspect, fork, or copy
ideas from — is untrusted input until you've read it. A skill file is a set of instructions an
agent may follow; a malicious or careless one can steer an agent toward leaking secrets, running
destructive commands, or installing something you didn't ask for.

Before adopting a third-party skill:

- Read the entire `SKILL.md` and any bundled `scripts/`/`hooks/` — don't skim.
- `grep` the skill directory for `curl`, `wget`, `bash`, `sh `, `rm -rf`, `sudo`, `chmod`,
  `postinstall`, `token`, `secret`, `kubectl apply`, `helm install`, `terraform apply`.
- Check who maintains it and whether it has a real commit/review history, not just a single
  drive-by commit.
- Clone it to a scratch directory first — never install sight-unseen via an automated installer.

## Avoid postinstall scripts and auto-executing installers

Skills should not require running an install script, a `postinstall` hook, or a `curl | bash`
pipe to work. If a skill collection bundles one, that's a signal to extract the markdown content
you want and skip the script — not to run it "because it's probably fine."

## Prefer markdown-only skills

A skill that's pure `SKILL.md` (+ optional `references/` markdown) has a much smaller attack
surface than one that ships `scripts/*.sh` the agent is expected to execute. Every skill in this
repository is markdown-only by design — see `docs/design-notes.md` for why.

## Review scripts before use

If you ever do add a skill that bundles a script, read it fully before letting an agent run it.
Pay particular attention to anything that shells out with `eval`, downloads and executes remote
content, or silently mutates infrastructure (`kubectl apply`, `helm install`, `terraform apply`,
`argocd app sync`) without a confirmation step.

## Never store secrets inside skills

Skills in this repository do not contain, request, or expect API keys, kubeconfigs, cloud
credentials, tokens, or any other secret material. If you customize a skill for your own
environment, keep credentials out of the skill file — reference an environment variable or
secrets manager by name, never a literal value.

## Never let a skill request credentials

No skill here should ask the user (or an agent) to paste in an API key, kubeconfig, or cloud
credential as part of its workflow. If you see a skill — third-party or your own — that does
this, treat it as a red flag and don't use it.

## Read-only commands first for infrastructure debugging

Every investigation/debug skill in this repository (`kubernetes-debug`, `argocd-debug`,
`terraform-review`, etc.) is written to prefer read-only commands (`kubectl get/describe/logs`,
`argocd app diff`, `terraform plan`/`validate`) and to treat anything that mutates live
infrastructure or state as requiring explicit user confirmation first.

## Dangerous commands require explicit user approval

No skill in this repository is designed to run `kubectl apply/delete`, `helm install/upgrade/
uninstall`, `terraform apply/destroy`, `argocd app sync`, or `git push` on its own. Every skill
that could plausibly lead to one of these commands stops and hands the exact command to the user
for review and manual execution, or asks for explicit confirmation first.

## Reporting a concern

If you find a skill in this repository (now or added later) that violates any of the above —
requests credentials, bundles an installer, or suggests running a destructive command without a
confirmation step — treat it as a bug and fix or remove it before using it further.
