# Tetragon deployment (eBPF runtime observability + enforcement)

Tetragon (a Cilium project) is the second sensor option beside Falco. Same domain, different
trade-off: policy evaluation and filtering happen **in-kernel** (low overhead at high event
rates) and it can **enforce** (kill a process at the syscall boundary), while Falco's strength
is its mature default ruleset and detection ecosystem.

## Falco vs Tetragon — decision table

| Dimension | Falco | Tetragon |
|---|---|---|
| Out-of-the-box detections | Large curated ruleset — value on day one | Few defaults; you author `TracingPolicy` objects for what you care about |
| Overhead model | Events stream to userspace for rule eval | In-kernel filtering — only matching events surface |
| Enforcement | Detect + alert (response via Falcosidekick/Talon) | Native: `Post` (observe) or **`Sigkill`/`Override`** action in-policy |
| Ecosystem fit | Standalone, any CNI | Natural beside Cilium (same project, shared labels/identity concepts) |
| Policy language | YAML rules + Sysdig filter syntax | `TracingPolicy` CRD: kprobes/tracepoints/LSM hooks + selectors |
| Kernel requirements | modern_ebpf: 5.8+ | BTF-enabled kernel (most distros ≥5.4 ship BTF; verify on homelab/custom kernels) |

Running both is legitimate (Falco for broad detection, Tetragon for targeted
observe-then-enforce policies) but doubles the privileged-DaemonSet surface — say why, or pick
one.

## Deployment shape

- Helm chart `tetragon` (Cilium repo): privileged DaemonSet + operator; values pin the export
  path. GitOps the chart + `TracingPolicy` objects like any other manifests.
- **Process lifecycle events come free** — every `process_exec`/`process_exit` with full
  ancestry, container/pod metadata, and (with Cilium) identity labels; this alone answers the
  "what spawned the shell" triage question without any policy authored.
- Export: JSON to stdout (scrape with the log pipeline), or gRPC → `tetra` CLI /
  Falcosidekick-style fan-out; feed the same alert routing as Falco (see
  `falco-deployment.md` §outputs) rather than building a second pipeline.

## TracingPolicy essentials

- Hook points: `kprobes` (syscalls — e.g. `security_file_open` for file access,
  `tcp_connect` for egress), `tracepoints`, LSM hooks. Prefer stable LSM/security_* hooks over
  raw syscall kprobes where possible (syscall ABI varies per arch/kernel).
- `selectors` filter in-kernel: by binary, args, pod label/namespace, in-init-tree, return
  value. The discipline from the main skill applies unchanged: exceptions scoped per-workload,
  never a policy deleted because it's noisy.
- Actions: `Post` (emit event — the default and the right first state for every policy),
  `Sigkill` (terminate the offending process), `Override` (fake a return value — block without
  kill).

## Enforcement discipline (the foot-gun)

`Sigkill` in a TracingPolicy is a production outage lever: a selector that's one label too
broad kills legitimate processes at kernel speed, cluster-wide, with no admission-controller
dry-run to save you.

- **Every enforcement policy ships as `Post` first**, runs long enough to see its real match
  set in the export stream, and only then graduates to `Sigkill` — same
  PERMISSIVE→observe→STRICT shape as mTLS rollout in the `istio` skill.
- Scope enforcement to the narrowest selector that expresses the invariant (specific binary in
  a specific namespace), never "any shell anywhere."
- Enforcement changes are reviewed like admission `failurePolicy: Fail` changes
  (`admission-policy-review`) — blast radius stated, rollback = revert to `Post`.
