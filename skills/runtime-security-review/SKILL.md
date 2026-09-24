---
name: runtime-security-review
description: Use when reviewing Kubernetes detection after admission — Falco/Tetragon/Tracee rules, runtime alerts, or suspicious container behavior. Mention "falco", "tetragon", "tracingpolicy", "tracee", or "runtime security"; use admission-policy-review for preventive admission rules and kubernetes-security for broad posture audits.
license: MIT
compatibility: Works with Claude Code, Codex-style agents, CodeWhale, OpenCode, and other agents that support Agent Skills-style instructions.
---

# Runtime Security Review

## When to use

Use when **deploying** Falco/Tracee (driver choice, DaemonSet, rule authoring, alert output),
reviewing runtime detection **coverage**, tuning noisy rules, triaging a runtime alert ("shell
spawned in container X"), or investigating suspicious behavior that build-time scanning can't
see. Completes the chain: `supply-chain-security` proves what was deployed,
`admission-policy-review` gates what runs, this watches what it *does*.
Use `admission-policy-review` instead for admission policy authoring and
`kubernetes-security` instead for a broad workload, RBAC, or cluster posture audit.

## Goal

Detection that fires on real compromise patterns with low enough noise to be pageable, plus a
triage path that distinguishes attack from ops-as-usual — without the agent ever mutating the
workload under investigation.

## Workflow

0. **Deployment (if not yet running)** — before coverage/triage there has to be a sensor. Plan
   the driver (`modern_ebpf` default, needs kernel 5.8+; `kmod` for bare metal; gVisor/managed-
   node caveats), the privileged DaemonSet, the k8s-metadata collector (so alerts name pods not
   container IDs), rule ConfigMaps in the GitOps repo, and the Falcosidekick output fan-out. Full
   mechanics, Helm values, rule syntax, and troubleshooting in `references/falco-deployment.md`.
   If the sensor choice is open (or Cilium is the CNI, or in-kernel *enforcement* is wanted),
   weigh Tetragon first — decision table and TracingPolicy/enforcement discipline in
   `references/tetragon-deployment.md`.

1. **Coverage check** — the detections that matter most, in rough order of signal value:
   - shell/exec into a container (`kubectl exec` legitimate use vs reverse shell — the alert
     needs the distinction, see tuning below)
   - package manager or compiler running in a production container (nothing should apt/pip/curl
     at runtime — immutable images make this a clean signal)
   - writes to `/etc`, `/bin`, `/usr` (should be impossible with readOnlyRootFilesystem — an
     alert here doubles as a hardening-gap detector)
   - access to the ServiceAccount token path from unexpected processes
   - outbound connections to unexpected destinations (miners, C2) — needs egress baseline
   - privilege escalation syscalls / capability use outside the pod's declared set
2. **Rule tuning discipline** — noise is the failure mode. Every rule review asks: what is the
   *legitimate* trigger of this event in this cluster (debug exec, init scripts writing config,
   backup agents reading volumes)? Exceptions are scoped per-workload (image, namespace, process
   ancestry), never global "disable rule"; each carries reason + owner like admission
   exceptions.
3. **Alert routing by confidence** — high-confidence rules (package manager in prod, write to
   /bin) can page; behavioral ones (outbound anomaly) go to a triage queue first. A runtime
   alert stream mixed 50/50 with ops noise trains on-call to ignore compromise.
4. **Triage a firing alert** — read-only, in order: which pod/namespace/image (is it even
   supposed to exist — cross-check against GitOps state); process tree from the alert payload
   (what spawned the shell — kubelet exec vs application process); recent events/deploys on
   that workload (rollout at the same timestamp usually explains it); image digest vs signed
   digests (unsigned/unknown image escalates immediately); network context if available.
5. **Contain deliberately** — containment (NetworkPolicy quarantine, scale-to-zero, node
   cordon) changes cluster state and destroys forensic evidence in the pod — it's a human
   decision. Propose the specific containment with tradeoffs; never execute it as part of
   triage. Capture evidence first where possible (logs, process list) since pod deletion loses
   everything ephemeral.
6. **Feed back** — every confirmed false positive becomes a scoped rule exception; every
   confirmed incident becomes an `incident-analysis` writeup and usually a new admission policy
   or hardening fix (the runtime alert that keeps firing is a control gap upstream).

## References

- `references/falco-deployment.md` — driver decision (modern_ebpf/ebpf/kmod, kernel floors,
  gVisor and managed-node caveats), privileged DaemonSet shape and Helm values, rule structure
  (lists/macros/rules, Sysdig filter fields, priorities), where rules live on nodes and how to
  override without forking (`append`), Falcosidekick output fan-out, k8saudit plugin, and
  deployment troubleshooting.
- `references/tetragon-deployment.md` — Falco vs Tetragon decision table, TracingPolicy hook
  points and in-kernel selectors, export wiring, and the Post→Sigkill enforcement discipline
  (observe first, narrowest selector, blast radius reviewed).

## Safety rules

- Investigation is read-only: `kubectl get/describe/logs`, alert payloads, audit logs. No
  `kubectl exec` into the suspect container (contaminates evidence and tips off an attacker),
  no pod deletion, no policy changes without explicit user decision.
- Never propose disabling a detection rule as the fix for noise — scope an exception instead,
  and say what coverage the exception removes.
- Treat "unknown image running in cluster" as an escalation to the user immediately, not a
  finding to batch into a report.

## Output format

For coverage review:
```
## Detections present vs the priority list
## Noise assessment (rules firing on ops-as-usual)
## Proposed rules/exceptions (scoped, with reason)
## Routing (page vs queue per rule)
```

For alert triage:
```
## Alert summary
## Evidence (read-only, quoted)
## Assessment: benign (why) | suspicious (what's unexplained) | confirmed hostile
## Proposed next step (containment options with tradeoffs — user decides)
```

## Quality checklist

- [ ] Every proposed exception is workload-scoped with a reason, never a global disable
- [ ] Alert triage cross-checked the workload against GitOps/signed state
- [ ] No exec into suspect containers, no mutation during investigation
- [ ] Containment was proposed with tradeoffs, not executed
- [ ] Confirmed findings feed back to admission policy / hardening, stated explicitly
