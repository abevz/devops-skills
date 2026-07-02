# Falco deployment and rule authoring

Self-authored reference (no third-party source). How Falco actually runs on nodes and in
Kubernetes, how rules are structured and where they live, and how alerts get out. This is the
"how it's set up" layer under the coverage/triage workflow in SKILL.md. Deployment-planning
content: propose values/manifests, never run `helm install` against a live cluster.

## How Falco sees syscalls: the driver decision

Falco needs a kernel-level source of syscall events. The driver is the first setup decision and
the most common source of "Falco won't start":

| Driver | Mechanism | Needs | Use when |
|---|---|---|---|
| `modern_ebpf` | CO-RE eBPF, embedded in Falco | Kernel **5.8+** with BTF | Default since 0.37 — no kernel headers, no compilation, works across kernels. First choice. |
| `ebpf` | Legacy eBPF probe (separate object) | BTF or a prebuilt/compiled probe | Older kernels that still have eBPF but < 5.8 |
| `kmod` | Loadable kernel module | Kernel headers to build, or a prebuilt module; privileged | Bare metal where eBPF is unavailable; node reboots need the module reloaded |

- ❌ `kmod` on managed node pools you don't control (GKE COS, EKS Bottlerocket, AKS) — you can't
  load arbitrary modules; use `modern_ebpf`.
- gVisor sandboxes (GKE Autopilot / runsc) don't expose host syscalls the normal way — Falco has
  a dedicated gVisor integration (reads runsc trace events); a standard driver sees nothing there.
- `falcoctl` (bundled) fetches/updates drivers and rules; in air-gapped clusters pre-stage the
  driver or the DaemonSet crash-loops on driver download.

Diagnose driver problems read-only: `kubectl logs -n falco ds/falco -c falco | grep -i driver`
and `... | grep -iE "Falco initialized|syscall|SCAP"`. "zero events" almost always = driver not
actually loaded.

## Kubernetes deployment shape

Falco runs as a **privileged DaemonSet** — one pod per node, because syscall capture is
per-kernel:

- `hostPID: true`, privileged securityContext (or a tuned capability set: `SYS_BPF`,
  `SYS_RESOURCE`, `SYS_PTRACE` for modern_ebpf), mounts `/proc`, `/dev`, `/etc`, host `/`.
  This is a deliberate, audited exception to your own hardening rules — Falco is the sensor, it
  needs the access. Note it explicitly in `kubernetes-security` reviews so it isn't flagged as a
  finding by mistake.
- Deploy via the `falcosecurity/falco` Helm chart. Key values:

```yaml
driver:
  kind: modern_ebpf          # modern_ebpf | ebpf | kmod
tty: true
falcosidekick:
  enabled: true              # fan-out for alerts (see Output section)
  config:
    slack: { webhookurl: "" }        # from a Secret, not inline
collectors:
  kubernetes:
    enabled: true            # enrich events with k8s.ns.name/k8s.pod.name (metadata plugin)
customRules:
  rules-local.yaml: |-       # your overrides — becomes a ConfigMap
    - macro: ...
```

- `collectors.kubernetes` (the `k8smeta`/container plugin) is what turns a raw PID into
  `k8s.ns.name=payments pod=api-xyz` — without it, alerts name a container ID nobody can map.
- Managed-cluster driver notes belong in the plan: GKE COS → modern_ebpf works; EKS Bottlerocket
  → modern_ebpf; AKS Ubuntu → any. Confirm kernel ≥ 5.8 for modern_ebpf on the actual node image.
- GitOps: the chart + custom-rules ConfigMap live in the GitOps repo like everything else; rule
  changes go through PR, not `kubectl edit` on the node.

## Rule structure — lists, macros, rules

Falco rules use three building blocks, evaluated with Sysdig filter syntax:

```yaml
- list: [name: allowed_shell_images, items: [debug-toolbox, ci-runner]]

- macro: shell_in_container
  condition: >
    spawned_process and container
    and proc.name in (bash, sh, zsh, ash)

- rule: Shell in container
  desc: Detect interactive shell spawned inside a container
  condition: shell_in_container and not container.image.repository in (allowed_shell_images)
  output: >
    Shell in container (user=%user.name container=%container.id
    image=%container.image.repository proc=%proc.cmdline
    k8s.ns=%k8s.ns.name k8s.pod=%k8s.pod.name)
  priority: WARNING
  tags: [container, shell, mitre_execution]
```

Common filter fields: `evt.type`, `proc.name`/`proc.cmdline`/`proc.pname` (parent),
`fd.name`/`fd.directory` (files/sockets), `container.id`/`container.image.repository`,
`user.name`/`user.uid`, `k8s.ns.name`/`k8s.pod.name`. Priorities:
`EMERGENCY..ALERT..CRITICAL..ERROR..WARNING..NOTICE..INFO..DEBUG` — the `priority` threshold in
falco.yaml (`priority: notice`) drops everything below it.

## Where rules live (nodes) and how to override without forking

| File | Purpose | Edit? |
|---|---|---|
| `/etc/falco/falco_rules.yaml` | Default ruleset (shipped) | ❌ never — overwritten on upgrade |
| `/etc/falco/falco_rules.local.yaml` | Your overrides/additions | ✅ |
| `/etc/falco/rules.d/*.yaml` | Additional rule files (dropped in) | ✅ |
| `/etc/falco/falco.yaml` | Engine config (driver, priority, outputs, `rules_files` load order) | ✅ |

Load order matters: later files override earlier by rule **name**. In Kubernetes these paths are
populated from the chart's `customRules` (ConfigMap mounted into the DaemonSet), so you never
`ssh` to a node — you change the ConfigMap via GitOps.

Tuning a noisy default rule without forking it: re-declare the macro it depends on with `append`,
or append an exception condition:

```yaml
- rule: Write below etc          # existing default rule name
  append: true
  condition: and not proc.name in (expected_config_writer)
```

This is the runtime equivalent of a scoped admission exception — narrow, named, reviewed. ❌
disabling the whole rule (`enabled: false`) to silence one legitimate trigger.

## Getting alerts out — Falcosidekick

Falco itself writes JSON to stdout/file/gRPC. **Falcosidekick** is the fan-out layer (deploy via
the chart's `falcosidekick.enabled`): one input, 50+ outputs. Typical wiring for this stack:

- `WARNING+` (real detections) → Alertmanager/PagerDuty (page or ticket per confidence, matching
  the routing logic in the SKILL.md workflow).
- All events → Loki/Elasticsearch for retention and correlation with app logs by `k8s.pod.name`.
- Slack/Teams for the triage channel.
- `minimumpriority` on outputs so INFO/DEBUG don't drown the page path.

Falcosidekick-UI gives a local event view; for real incident work correlate in
Loki/Grafana against deploy annotations (ties into `observability-review`).

## Falco plugins (beyond syscalls)

The syscall source is one input. Plugins add others: `k8saudit` (Kubernetes API audit log —
detect `exec`/`attach`/RBAC changes at the API layer, which syscalls can't see), `cloudtrail`
(AWS API activity), `github`, `okta`. `k8saudit` is the high-value second source for this stack —
it catches "someone `kubectl exec`'d into prod" at the API server even when the syscall-level
shell rule is tuned down.

## Deployment troubleshooting

| Symptom | Cause / check |
|---|---|
| DaemonSet CrashLoopBackOff on start | Driver load failure — `kubectl logs` for driver/BTF errors; wrong driver `kind` for the node kernel; falcoctl can't fetch driver (air-gapped) |
| Falco running, zero alerts ever | Driver loaded but wrong, or `priority` threshold too high; verify with a benign trigger (`kubectl exec` into a test pod, expect the shell rule) |
| Alerts name container IDs, not pods | `collectors.kubernetes`/k8s metadata plugin disabled |
| Storm of alerts on deploy | Default rules firing on ops-as-usual — tune with `append` exceptions, don't disable |
| High CPU on nodes | Ruleset too broad / very high event rate; scope rules, raise priority threshold, or exclude high-churn namespaces |
| Works on some nodes, not others | Mixed node images/kernels — a node pool below the modern_ebpf kernel floor |
