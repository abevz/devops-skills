---
name: homelab-change-plan
description: Use when planning a change to homelab infrastructure such as Proxmox VMs, a self-hosted Kubernetes cluster, or IaC-managed home services, where there is no staging environment. Mention "homelab change", "upgrade my proxmox", "change to my home cluster" as triggers.
license: MIT
compatibility: Works with Claude Code, Codex-style agents, CodeWhale, OpenCode, and other agents that support Agent Skills-style instructions.
---

# Homelab Change Plan

## When to use

Use before making a non-trivial change to homelab infrastructure — Proxmox host or VM changes,
self-hosted Kubernetes upgrades, IaC refactors (OpenTofu/Terraform + Ansible), storage or
network changes — where production and staging are the same machines and the operator is one
person.

## Goal

Produce a change plan adapted to homelab reality: no staging, no on-call team, shared hardware,
and services you actually depend on (mail, Git, DNS, monitoring) — so the plan leans harder on
snapshots, backups, and reversibility than a work migration plan would.

## Workflow

1. **Classify the blast radius first** — homelab services aren't equal. Tier the affected
   services: (a) *depended-on daily* (mail, Git/GitLab, DNS, password manager, monitoring),
   (b) *annoying to lose* (media, dashboards), (c) *lab/experimental*. A change touching tier
   (a) gets the full plan; tier (c) can move fast.
2. **Check the escape hatch before starting** — verify backups/snapshots actually exist and are
   restorable *before* the change, not during the incident: Proxmox snapshot or vzdump for
   affected VMs, etcd/velero for K8s state, config in git, database dumps for stateful services.
   "Backup exists" means a restore has been tested at least once, or the plan says it hasn't.
3. **Order the change to keep the recovery path alive** — don't take down the thing you'd need
   to fix a failure: the hypervisor hosting your only DNS, the monitoring that tells you what
   broke, the Git server holding the IaC that rebuilds everything. If the change touches these,
   plan an out-of-band fallback first (hosts-file entries, local IaC checkout, console access).
4. **Prefer IaC-shaped changes** — if the infrastructure is OpenTofu/Ansible-managed, the change
   goes through code (plan → review → apply), not ad hoc console clicks that create drift.
   One-off manual changes get a follow-up task to encode them back into IaC.
5. **Sequence in reversible steps** — snapshot → change one thing → verify → next. For host-level
   changes (Proxmox upgrades, kernel, storage), note the point of no return explicitly (e.g.
   storage format migration) and front-load everything reversible before it.
6. **Verify like production** — a concrete check per affected service (not "it boots"): mail
   flows, DNS resolves, cluster nodes Ready, monitoring targets up, backup jobs green on the
   next cycle.
7. **Secrets discipline** — any secrets touched during the change stay in the existing secrets
   mechanism (SOPS/Vault/sealed-secrets); nothing gets pasted into shell history, IaC files, or
   notes "temporarily."
8. **Time-box and schedule** — homelab changes happen in personal time; estimate honestly,
   schedule when a failure won't hurt (not before a workday that needs the mail server), and
   define the bail-out condition: "if not working by X, restore the snapshot and stop."

## Safety rules

- Do not execute the change while planning it — this skill produces the plan; execution is a
  separate, deliberate step.
- Never plan a change to tier-(a) services without a verified, restorable backup — if the
  restore path is untested, the plan must say so in bold rather than assume it works.
- Do not propose ad hoc manual changes to IaC-managed infrastructure without a step that
  reconciles the drift back into code.
- Never include actual secret values in the plan — reference where they live, not what they are.

## Output format

```
## Change summary
## Affected services by tier
<(a) depended-on / (b) annoying / (c) lab>
## Escape hatch status
<backups/snapshots: exist? restore tested?>
## Recovery-path check
<what I'd need if this fails, and whether the change takes it down>
## Steps
1. <step> — verify: <check> — revert: <how>
## Point of no return
## Verification per service
## Bail-out condition & time box
```

## Quality checklist

- [ ] Services are tiered and the plan depth matches the highest tier touched
- [ ] Backup/snapshot restorability is verified or explicitly flagged as untested
- [ ] The recovery path (DNS/monitoring/IaC repo/console) survives every step
- [ ] Every step has a verify and a revert
- [ ] The point of no return and bail-out condition are explicit
