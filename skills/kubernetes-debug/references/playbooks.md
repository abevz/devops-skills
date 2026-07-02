# Failure-mode playbooks

Symptom-keyed diagnosis trees. Jump to the section matching the observed state; each maps the
exact evidence (event text, exit code, condition) to its cause and fix. All diagnosis commands
are read-only.

## Pod Pending

`kubectl describe pod <pod>` → read the Events section. The scheduler tells you exactly why:

| Event text contains | Cause | Verify / fix |
|---|---|---|
| `0/N nodes are available: insufficient cpu` (or `memory`) | Requests don't fit any node | `kubectl describe nodes \| grep -A5 Allocated` — compare against pod requests; lower requests or add capacity |
| `exceeded quota` | Namespace ResourceQuota exhausted | `kubectl describe resourcequota -n <ns>` — raise quota or free resources |
| `had untolerated taint {...}` | Node taints, pod lacks toleration | `kubectl get nodes -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints` — add toleration or use untainted pool |
| `didn't match Pod's node affinity/selector` | nodeSelector/affinity matches no node | Check labels: `kubectl get nodes --show-labels` vs the pod's nodeSelector |
| `pod has unbound immediate PersistentVolumeClaims` | PVC not bound | `kubectl get pvc -n <ns>` → if Pending, `describe pvc`: missing StorageClass, provisioner down, or no capacity |
| `volume node affinity conflict` | PV is zonal, pod scheduled to another zone | Check PV nodeAffinity vs pod constraints; usually needs `WaitForFirstConsumer` binding mode |
| **No events at all** | Scheduler not seeing it: `schedulingGates`, custom schedulerName, or scheduler down | `kubectl get pod -o yaml \| grep -A3 schedulingGates`; check schedulerName |

## CrashLoopBackOff

First: `kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[*].lastState.terminated}'`
— the `exitCode` and `reason` route everything:

| exitCode / reason | Meaning | Next step |
|---|---|---|
| `reason: OOMKilled` (usually 137) | Memory limit exceeded | See OOMKilled section below |
| 137, reason `Error` | SIGKILL not from OOM: liveness probe kill or eviction | `describe pod` → probe failure events; probe too aggressive or app genuinely unhealthy |
| 143 | SIGTERM received and exited | Normal shutdown signal — who sent it? Rollout, probe, preStop issues |
| 1 (or app-specific) | App exited with error | `kubectl logs <pod> --previous` — the real error is in the *previous* container's logs |
| 0 | Process finished "successfully" and restartPolicy restarted it | Long-running service whose main process daemonizes/exits (run in foreground), or this should be a Job |
| 126 | Command found but not executable | Wrong permissions/format on entrypoint (e.g. script without shebang or exec bit) |
| 127 | Command not found | Wrong `command:`/`args:` or binary missing from image — check image contents and manifest overrides |
| 139 | Segfault | App/native-lib bug; check arch mismatch (arm64 image on amd64 node shows as exec format error in logs) |

Not actually a crash:

- `CreateContainerConfigError` → referenced ConfigMap/Secret or key doesn't exist:
  `describe pod` names the missing object exactly.
- `RunContainerError` with `exec format error` → wrong-architecture image.
- Restart counter climbing with `Running` state → liveness probe killing a healthy-but-slow app:
  compare probe `initialDelaySeconds`/`failureThreshold` against real startup time; consider a
  `startupProbe`.
- Init container failing → `kubectl logs <pod> -c <init-container>` — the main container never
  gets a chance.

## ImagePullBackOff / ErrImagePull

`kubectl describe pod` → the pull error text is diagnostic:

| Error text contains | Cause | Fix |
|---|---|---|
| `unauthorized` / `authentication required` | Missing/wrong `imagePullSecrets`, expired registry creds | Check secret exists in *this* namespace, is type `kubernetes.io/dockerconfigjson`, and is referenced by the pod/SA |
| `not found` / `manifest unknown` | Tag or repo doesn't exist | Typo, tag deleted/never pushed, wrong registry path |
| `toomanyrequests` | Docker Hub rate limit | Authenticated pulls, mirror, or move image to your registry |
| `i/o timeout` / `no such host` | Node can't reach registry | Node-level DNS/proxy/firewall; test from the node, not your laptop |
| `x509: certificate signed by unknown authority` | Private registry with untrusted CA | CA must be trusted by the container runtime on nodes |

## OOMKilled

1. Confirm: `lastState.terminated.reason: OOMKilled`.
2. Pattern check — `kubectl top pod` over time (or memory graph): memory climbs steadily to the
   limit and resets on restart = leak or unbounded cache; spikes on specific requests = payload-
   driven; killed at startup = limit simply below the app's baseline.
3. Runtime awareness: JVM before container-aware flags, or Go without `GOMEMLIMIT`, will happily
   overshoot — fix the runtime config, not just the limit.
4. Raising the limit is a mitigation, not a fix, unless baseline genuinely grew — say which one
   applies.

## Service has endpoints but no traffic / connection refused

Walk the chain; each link has one check:

1. **Selector → pods**: `kubectl get endpointslices -l kubernetes.io/service-name=<svc> -n <ns>`
   — empty? Service selector doesn't match pod labels (compare exactly), or no pod is Ready.
2. **Ready?** Pods can Run but not be Ready — readiness probe failing removes them from
   endpoints. `kubectl get pods -o wide` READY column.
3. **Port mapping**: Service `targetPort` must match the *container's actual listening port*
   (and if `targetPort` is a name, the containerPort must carry that name).
4. **Listening address**: app binding `127.0.0.1` inside the container is unreachable via the
   pod IP — must bind `0.0.0.0`. (`kubectl exec` + `ss -lnt` if a shell exists.)
5. **NetworkPolicy**: `kubectl get netpol -n <ns>` — a default-deny without a matching allow
   silently drops; remember both directions and both namespaces.
6. **Type-specific**: NodePort needs node firewall open; LoadBalancer needs the LB to be
   provisioned (`kubectl get svc` EXTERNAL-IP not `<pending>`).

## DNS resolution failures

1. Reproduce from a pod in the same namespace: `kubectl run dnstest --rm -it --image=busybox
   --restart=Never -- nslookup <name>` (mutating but ephemeral — ask first, or use an existing
   pod with a shell).
2. Short name vs FQDN: `svc` resolves within the namespace; cross-namespace needs
   `svc.<ns>.svc.cluster.local` (or at least `svc.<ns>`).
3. CoreDNS itself: `kubectl get pods -n kube-system -l k8s-app=kube-dns` +
   `kubectl logs -n kube-system -l k8s-app=kube-dns` — look for upstream failures/loops.
4. NetworkPolicy blocking UDP/TCP 53 egress to kube-dns — the classic side effect of a
   default-deny egress policy.
5. Intermittent slow lookups (5s pattern) → `ndots:5` + external names: every external lookup
   tries cluster suffixes first; fix with FQDN (trailing dot) or dnsConfig.

## Latency spikes without errors (CPU throttling)

CFS throttling from CPU limits is invisible in error rates — the app just gets slower in bursts:

1. Signature: p99 latency spikes, no errors, no OOMKills, CPU "usage" below the limit on
   dashboards (throttling happens within averaging windows).
2. Confirm: container_cpu_cfs_throttled_periods_total / container_cpu_cfs_throttled_seconds_total
   metrics for the pod, or `kubectl exec` → `cat /sys/fs/cgroup/cpu.stat` (`nr_throttled`).
3. Fix: raise or remove the CPU limit (keep the request); CPU limits are for multi-tenant
   fairness and batch, not a default for latency-sensitive services.

## Whole service restart-looping at once (cascading liveness failure)

All replicas restarting simultaneously = liveness probe checking an external dependency:

1. Signature: every pod of the service restarts within the same window; a shared dependency
   (DB, cache) had an incident at the start of it.
2. Confirm: `kubectl get events --field-selector reason=Unhealthy` timing vs the dependency
   outage; read the liveness endpoint's implementation — does it ping the DB?
3. Fix: liveness checks process health only; dependency checks move to readiness (pod leaves
   endpoints but survives). This turns "DB blip = full service crash loop" into "DB blip =
   temporary traffic pause."

## Node NotReady

`kubectl describe node <node>` → Conditions table:

- `MemoryPressure`/`DiskPressure`/`PIDPressure: True` → kubelet is evicting; find the consumer
  (`kubectl top node`, disk: image garbage, logs).
- `Ready: Unknown` (with `NodeStatusUnknown`) → kubelet stopped reporting: node down, kubelet
  crashed, or network partition to control plane.
- CNI errors in events (`network plugin not ready`) → CNI agent pod on that node is the patient,
  not the node.
- After it recovers, check what was evicted and whether it rescheduled cleanly.
