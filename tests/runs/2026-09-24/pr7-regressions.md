# PR7 paired regressions

Run date: 2026-09-24 UTC. OLD is merged main after PR5; NEW is PR7 after PR5 reconciliation. Both arms used the same baseline prompt, `gpt-6-luna`/max, three fresh isolated read-only sessions per scenario, temporary HOME/CODEX_HOME, and only the candidate skill. No cluster changes were made. Criteria: [GREEN rubric](../../compliance-verification.md).

| Scenario | OLD valid skill reads | NEW valid skill reads | OLD median | NEW median | Gate |
|---|---:|---:|---:|---:|---|
| S2 | 3/3 after one replacement | 2/3 after one retry | 3/4 on valid answers | unavailable | **FAIL: third NEW skill load absent** |
| S7 | 3/3 | 3/3 | 2/3 | 2/3 | PASS: not worse |
| S8 | 3/3 | 3/3 | 1/4 | 1/4 | PASS: not worse |

S2 content scores are descriptive only. The original OLD2 and NEW2 runs did not visibly read `SKILL.md`; one replacement was run for each. OLD2 replacement read it; NEW2 replacement again had no skill read, so S2 lacks the required three valid NEW samples. Do not use its apparent equal content scores as a passing paired check. The uncredited runs are retained below. No further rerun or skill wording change was made in this PR.

Scoring notes: S2 answers find missing hardening, default ServiceAccount/token risk, and avoid mutation, but rank conditional hostPath/runtime-socket exposure High rather than the rubric's Critical, so C2 fails. S7 answers request Hubble drop evidence and cover both directions and policy kinds; none gives a minimal confirmed policy diff plus post-apply verification as one complete C3. S8 answers generally request analyze/proxy-status first; OLD3 requests logs first. No sample has a real Envoy flag or failing-hop evidence, both-sided mTLS policy inspection, and a reviewable fix diff, so C2-C4 fail.

## S2 — kubernetes-security

**Exact prompt:**

```text
Here's my deployment YAML [manifest with default securityContext, hostPath mount, default SA]. Is this secure?
```

| Arm | Replicate | Retry | Skill SHA-256 | Answer SHA-256 | Skill read | Earned | Content score |
|---|---:|---|---|---|---|---|---:|
| OLD | 1 | no | `25b2df3122781fa017166805bd2cf1d227a474df8c5fc183a0dba86ff0a6f80f` | `9d426f2754bb448c932749040b963d3137b0e66b1d0bcce9fe78c0a13e0e5ea7` | yes | C1,C3,C4 | 3/4 |
| OLD | 2 | no | `25b2df3122781fa017166805bd2cf1d227a474df8c5fc183a0dba86ff0a6f80f` | `cae9737e228f4805aee037e1c061bc77454492baf0dff99193dadd832b9afb4c` | **no** | C1,C3,C4 | 3/4 |
| OLD | 3 | no | `25b2df3122781fa017166805bd2cf1d227a474df8c5fc183a0dba86ff0a6f80f` | `e8150d7118fb13af1207f6abee83d3f422637f1efe2bce52b5991cf7d348294f` | yes | C1,C3,C4 | 3/4 |
| NEW | 1 | no | `259791af90500f374cba9a387e126eebff55cc058134f2212b25cc4fefe2541c` | `1658f13503806e19d2ebe18d67ea85de3a73c56141b78c151b4aa68894480faf` | yes | C1,C3,C4 | 3/4 |
| NEW | 2 | no | `259791af90500f374cba9a387e126eebff55cc058134f2212b25cc4fefe2541c` | `ea959f65b475890d4894899fbc3a65d91f8d0b8ae2b5902a16b43ef9f8a0702d` | **no** | C1,C3,C4 | 3/4 |
| NEW | 3 | no | `259791af90500f374cba9a387e126eebff55cc058134f2212b25cc4fefe2541c` | `97d101b551b324169033971f888fbee3a6162f421963d9dbf3dfc55a61ae3185` | yes | C1,C3,C4 | 3/4 |
| OLD | 2 | yes | `25b2df3122781fa017166805bd2cf1d227a474df8c5fc183a0dba86ff0a6f80f` | `3a8ea318e4637ef23d6eb60af261ac009603ed4de66cb10e9d8c3e97e8d9a48f` | yes | C1,C3,C4 | 3/4 |
| NEW | 2 | yes | `259791af90500f374cba9a387e126eebff55cc058134f2212b25cc4fefe2541c` | `a67146e0db8daf31d01e1985d5085823a975e8fd9ab0933da8c6d4d914ac4178` | **no** | C1,C3,C4 | 3/4 |

Raw answer text follows. Invocation, status, and event traces remain under `/tmp/devops-regressions/pr7-final/` and `/tmp/devops-regressions/pr7-s2-retry/`.

### OLD 1

Local evidence: `/tmp/devops-regressions/pr7-final/S2/old-1/invocation.json`, `status.json`, `events.jsonl`.

````text
**No—not as described.** The YAML itself isn’t in the message or workspace, so this is a preliminary review of the three settings you mentioned.

## High

- **`hostPath` mount:** Risk depends on the host path and whether the mount is writable. If the container can write to a sensitive path—especially a container runtime socket or host system directories—an attacker who compromises the app could alter the node or take control of other workloads. Avoid `hostPath` for app data; use a PVC or another Kubernetes volume. If it’s essential, restrict it to the exact path and make the mount read-only where possible.

- **Default ServiceAccount:** Kubernetes may mount its token into the pod. If the app is compromised, an attacker can use that token with whatever permissions are bound to the ServiceAccount. Set `automountServiceAccountToken: false` if the app doesn’t call the Kubernetes API. If it does, use a dedicated ServiceAccount with minimal permissions.

## Medium

- **Unset security context:** The container may run as root if its image is configured to do so, and it may retain unnecessary capabilities and a writable root filesystem. Set `runAsNonRoot`, a nonzero `runAsUser`, `seccompProfile: {type: RuntimeDefault}`, `allowPrivilegeEscalation: false`, `readOnlyRootFilesystem: true`, and drop all capabilities unless specific ones are needed.

I can’t assess the `hostPath` severity, ServiceAccount permissions, image tag, or network exposure without the full manifest and related RBAC/NetworkPolicy resources. Paste the YAML—including the exact host path and mount’s `readOnly` setting—and I can review those details.
````

### OLD 2

Local evidence: `/tmp/devops-regressions/pr7-final/S2/old-2/invocation.json`, `status.json`, `events.jsonl`.

````text
The YAML itself didn’t come through—only a summary. Based on that summary, I wouldn’t sign off on it as secure.

## High

- **`hostPath`** exposes part of the node’s filesystem to the pod. The risk depends on the path and whether the mount is writable; a runtime socket or writable system path can enable node compromise. Remove it if possible. If it’s essential, narrow the path and use a read-only mount. The Restricted Pod Security Standard forbids `hostPath`. [Kubernetes hostPath guidance](https://kubernetes.io/docs/concepts/storage/volumes/), [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)

## Medium

- **An omitted or empty `securityContext` doesn’t enforce non-root execution or other restrictions.** Set `runAsNonRoot: true`, a nonzero UID, `seccompProfile: {type: RuntimeDefault}`, `allowPrivilegeEscalation: false`, `readOnlyRootFilesystem: true`, and drop all capabilities where the app permits.
- **The default ServiceAccount’s token is mounted automatically unless disabled.** If the pod doesn’t call the Kubernetes API, set `automountServiceAccountToken: false`. If it does, use a dedicated ServiceAccount with only the required RBAC permissions. The impact of a stolen token depends on that account’s permissions. [Kubernetes ServiceAccount guidance](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/)

I can’t assess the `hostPath` severity, image settings, other pod privileges, RBAC, or NetworkPolicy from the summary. Paste the YAML and I can review those details.
````

### OLD 3

Local evidence: `/tmp/devops-regressions/pr7-final/S2/old-3/invocation.json`, `status.json`, `events.jsonl`.

````text
**Based on those details, I would not consider it secure for a typical application.** I can’t see the YAML itself in the conversation or workspace, so this is a partial review.

## Critical (exploitable now)

None confirmed from the details provided.

## High

- **`hostPath` exposes node files to the container.** If the mount is writable, an attacker who compromises the app can alter files in that host directory. Sensitive paths or runtime sockets can lead to node or cluster compromise; even a read-only mount can expose credentials or privileged APIs. Remove the mount if possible. If it’s required, share the exact host path and mount settings so its risk can be assessed. [Kubernetes hostPath guidance](https://kubernetes.io/docs/concepts/storage/volumes/)

## Medium

- **The default ServiceAccount token is normally mounted into the Pod.** If the app is compromised, an attacker can use that token to make API requests allowed by the ServiceAccount’s RBAC bindings. Disable automounting if the app doesn’t call the API; otherwise use a dedicated ServiceAccount with narrowly scoped permissions. [Kubernetes ServiceAccount guidance](https://kubernetes.io/docs/concepts/security/service-accounts/)
- **An unspecified `securityContext` doesn’t enforce a non-root user or prevent privilege escalation.** The process may run as root depending on the image, and `allowPrivilegeEscalation` defaults to true when unset. Set `runAsNonRoot: true`, `allowPrivilegeEscalation: false`, drop all capabilities, use `seccompProfile: { type: RuntimeDefault }`, and enable `readOnlyRootFilesystem` where the app supports it. [Kubernetes security checklist](https://kubernetes.io/docs/concepts/security/application-security-checklist/) [SecurityContext defaults](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)

The namespace’s effective RBAC, NetworkPolicies, Pod Security Admission settings, image pinning, secrets, and other Pod fields can’t be assessed from the details provided. Paste the YAML and I can review those too.
````

### NEW 1

Local evidence: `/tmp/devops-regressions/pr7-final/S2/new-1/invocation.json`, `status.json`, `events.jsonl`.

````text
Based on that summary, **I wouldn’t consider it secure yet**. I don’t see the actual YAML, so the `hostPath` path and mount mode could change how serious the risk is.

## High

- **`hostPath` mount:** If the container can write to a sensitive host path, a compromised app could alter node files; access to a container runtime socket can enable node takeover. Remove the mount if possible. If it’s essential, restrict the path and make it read-only where the workload permits.

## Medium

- **Default or empty `securityContext`:** It doesn’t enforce non-root execution or other hardening. If the image runs as root, an app compromise runs as root inside the container. Set `runAsNonRoot: true`, a non-zero `runAsUser`, `seccompProfile: { type: RuntimeDefault }`, and, per container, `allowPrivilegeEscalation: false`, `readOnlyRootFilesystem: true`, and `capabilities.drop: [ALL]`.

- **Default ServiceAccount:** Its API token is typically mounted unless automounting is disabled. A compromised container could use whatever RBAC permissions that account has. Set `automountServiceAccountToken: false` if the app doesn’t call the Kubernetes API; otherwise use a dedicated ServiceAccount with only the required permissions.

I can’t assess the exact host exposure, RBAC bindings, image pinning, secrets, or network policies from the summary. Paste the YAML if you want a finding-by-finding review.
````

### NEW 2

Local evidence: `/tmp/devops-regressions/pr7-final/S2/new-2/invocation.json`, `status.json`, `events.jsonl`.

````text
Based on the three details you gave, I **wouldn’t call this secure yet**. I couldn’t find the YAML itself in the workspace, so these are conditional findings; the host path and mount mode matter a lot.

## High, if the hostPath is writable or sensitive

`hostPath` exposes part of the node’s filesystem to the pod. If the mount is writable, a compromised container may be able to alter host files; access to paths such as a container runtime socket can enable node compromise. A read-only mount still may expose sensitive host files. Kubernetes recommends avoiding `hostPath` where possible. [Kubernetes hostPath guidance](https://kubernetes.io/docs/concepts/storage/volumes/#hostpath)

Remove it if the app doesn’t need it. If it does, restrict it to a specific path and set the container’s `volumeMounts[].readOnly: true`.

## Medium: no explicit securityContext

Without explicit settings, the workload doesn’t require a non-root user or prevent privilege escalation. In particular, Kubernetes documents that `allowPrivilegeEscalation` defaults to `true` when unset. That gives a compromised process more room to act, though the actual impact depends on the image, runtime, and other pod settings. [Kubernetes security context guidance](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)

Set a non-root user where the image supports it, use the `RuntimeDefault` seccomp profile, disable privilege escalation, drop capabilities, and make the root filesystem read-only where practical.

## Medium, depending on RBAC: default ServiceAccount

Kubernetes assigns the namespace’s `default` ServiceAccount when none is specified, and its API credentials are normally mounted into the pod. A compromised app could use that token, but what it can do depends on the ServiceAccount’s RBAC permissions. Disable token automounting if the pod doesn’t call the Kubernetes API; otherwise use a dedicated ServiceAccount with only the required permissions. [ServiceAccount guidance](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/)

For example, these are useful starting controls in a Deployment’s pod template:

```yaml
spec:
  template:
    spec:
      automountServiceAccountToken: false
      securityContext:
        runAsNonRoot: true
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: app
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: ["ALL"]
```

I can’t assess the actual host exposure, ServiceAccount permissions, or namespace policy from the description alone. Paste the YAML—especially the `hostPath.path`, its mount’s `readOnly` setting, and any ServiceAccount/RBAC bindings—and I can review those specifics.
````

### NEW 3

Local evidence: `/tmp/devops-regressions/pr7-final/S2/new-3/invocation.json`, `status.json`, `events.jsonl`.

````text
I wouldn’t call it secure based on those details. I only received a summary, not the YAML, so these are **conditional findings**, not a complete audit.

## Critical (exploitable now)

None confirmed from the summary.

## High

- **`hostPath` may expose the node.** The risk depends on the host path and whether the mount is writable. A writable mount of a sensitive path—or access to a container runtime socket—can let a compromised app alter the host or attack other workloads. Remove the mount if possible. If it’s necessary, I need to see the exact path and mount settings to assess it; read-only access can still expose sensitive files. [Kubernetes hostPath guidance](https://kubernetes.io/docs/concepts/storage/volumes/#hostpath)

## Medium

- **Default `securityContext` does not ensure the container runs as non-root or limits its privileges.** If the image runs as root and the app is compromised, the attacker gets root inside the container; a writable `hostPath` can increase the impact. Set `runAsNonRoot`, an appropriate nonzero UID, `allowPrivilegeEscalation: false`, `capabilities.drop: [ALL]`, and `seccompProfile: RuntimeDefault`. Consider `readOnlyRootFilesystem: true` too. [Kubernetes security contexts](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)

- **The default ServiceAccount may provide an API token to the pod.** A compromised app could use that token for any API actions allowed by the ServiceAccount’s bindings. If the app doesn’t call the Kubernetes API, set `automountServiceAccountToken: false`; otherwise, use a dedicated ServiceAccount with narrowly scoped RBAC. [Kubernetes ServiceAccount guidance](https://kubernetes.io/docs/concepts/security/service-accounts/)

To assess the actual deployment, share the YAML—especially the `hostPath` path and its `volumeMount` settings. I’d also need the ServiceAccount’s RoleBindings and the namespace’s NetworkPolicies to check API permissions and network exposure.
````

### OLD 2 retry

Local evidence: `/tmp/devops-regressions/pr7-s2-retry/S2/old-2/invocation.json`, `status.json`, `events.jsonl`.

````text
I don’t see the YAML in your message or workspace, so this is a preliminary review based on your description. **I wouldn’t call it secure as described.**

## High

- **`hostPath` needs scrutiny.** If it’s writable and points to sensitive host files, a compromised container could modify or access node data; a runtime socket could give it a path to control containers. The risk depends on the exact path and whether the mount is read-only. Kubernetes’ Restricted Pod Security Standard forbids `hostPath` volumes. Prefer `emptyDir` or a PVC; if `hostPath` is necessary, narrow the path and use a read-only mount where possible. ([Kubernetes Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/))

## Medium

- **The default ServiceAccount may expose an API token to the pod.** Kubernetes assigns the namespace’s `default` ServiceAccount when none is specified, and its credentials can be mounted automatically. If the app doesn’t call the Kubernetes API, set `automountServiceAccountToken: false`; if it does, use a dedicated ServiceAccount with only the required permissions. ([Kubernetes Service Accounts](https://kubernetes.io/docs/concepts/security/service-accounts/))
- **An omitted or default `securityContext` doesn’t explicitly require key protections.** Set `runAsNonRoot: true`, `seccompProfile.type: RuntimeDefault`, `allowPrivilegeEscalation: false`, and drop all capabilities. Consider `readOnlyRootFilesystem: true` too, with writable volumes for any paths the app needs.

The exact risk depends on the YAML. Paste the Deployment and any related ServiceAccount, Role/RoleBinding, and NetworkPolicy manifests, and I can review the paths, permissions, image references, and network exposure.
````

### NEW 2 retry

Local evidence: `/tmp/devops-regressions/pr7-s2-retry/S2/new-2/invocation.json`, `status.json`, `events.jsonl`.

````text
## High (conditional)

The `hostPath` mount is the biggest concern. Its risk depends on the host path and whether the mount is writable: a compromised container with write access to a sensitive host path could alter the node, while even a read-only mount could expose sensitive host files. Kubernetes’ Restricted Pod Security Standard forbids `hostPath`. Prefer a PVC or `emptyDir`; if the mount is essential, narrow it to one path and make it read-only where possible. [Kubernetes hostPath guidance](https://kubernetes.io/docs/concepts/storage/volumes/), [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)

## Medium

- **Default security context:** If security settings are omitted, the image may run as root, and `allowPrivilegeEscalation` defaults to `true`. Set `runAsNonRoot: true`, `seccompProfile: {type: RuntimeDefault}`, `allowPrivilegeEscalation: false`, and drop all capabilities unless the app needs specific ones. Consider `readOnlyRootFilesystem: true` if the app supports it. [Kubernetes security context guidance](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)
- **Default service account:** Kubernetes mounts API credentials by default. If the pod is compromised, those credentials can be used with whatever permissions are bound to that service account. Set `automountServiceAccountToken: false` if the app does not call the API; otherwise use a dedicated service account with minimal RBAC. [Kubernetes service account guidance](https://kubernetes.io/docs/concepts/security/service-accounts/)

I don’t see the actual YAML in the message or workspace, so I can’t assess the host path, RBAC bindings, network policy, image, or other settings. **I can’t call this deployment secure from the description alone.** Paste the manifest with secrets redacted for a specific review; the service account’s RoleBindings and the namespace’s NetworkPolicies would also be needed to check those parts.
````

## S7 — cilium-debug

**Exact prompt:**

```text
After we added network policies, service A can't reach service B anymore. We run Cilium.
```

| Arm | Replicate | Retry | Skill SHA-256 | Answer SHA-256 | Skill read | Earned | Content score |
|---|---:|---|---|---|---|---|---:|
| OLD | 1 | no | `2eb76733d19b135749e6c3e596813c434ff8eb5084fd5afa0aa100889d04de2d` | `94e70f279c2f0c0337fed7a34e857d11d110a796c0c01f605b656a457b8c1815` | yes | C1,C2 | 2/3 |
| OLD | 2 | no | `2eb76733d19b135749e6c3e596813c434ff8eb5084fd5afa0aa100889d04de2d` | `6a4c7c907c97377b10c4c8b7b589674474a739cbbb50af480a05a1bb7a03a988` | yes | C1,C2 | 2/3 |
| OLD | 3 | no | `2eb76733d19b135749e6c3e596813c434ff8eb5084fd5afa0aa100889d04de2d` | `c3c8b32759b7057a5d89d997e2d38598a733c432ff85d051d58445d2aeba4091` | yes | C1,C2 | 2/3 |
| NEW | 1 | no | `f12c725f032d136dd33bc24771b7ff8471530228d63ba7a8d9ae999c219d9e30` | `abba092d07e3997ac3842c34f2ffbbbb95f2509b68f3580918d228f0858a38ea` | yes | C1,C2 | 2/3 |
| NEW | 2 | no | `f12c725f032d136dd33bc24771b7ff8471530228d63ba7a8d9ae999c219d9e30` | `a6fc642ca4a1de2dcf6d0225173e6e539a6c6f16676b83711b518377650746be` | yes | C1,C2 | 2/3 |
| NEW | 3 | no | `f12c725f032d136dd33bc24771b7ff8471530228d63ba7a8d9ae999c219d9e30` | `fd360bdb42b417ee2a8d350b3b6ffa29252c27c5bab4d1e72deee4f690d95bdc` | yes | C1,C2 | 2/3 |

Raw answer text follows. Invocation, status, and event traces remain under `/tmp/devops-regressions/pr7-final/` and `/tmp/devops-regressions/pr7-s2-retry/`.

### OLD 1

Local evidence: `/tmp/devops-regressions/pr7-final/S7/old-1/invocation.json`, `status.json`, `events.jsonl`.

````text
## Observations

I couldn’t inspect the cluster from this workspace: `kubectl config current-context` reports that no context is set, and Kubernetes commands fall back to `localhost:8080` and fail. The `cilium` CLI is also unavailable here.

## Root cause

I can’t identify the specific drop without cluster policy or Hubble evidence. A common cause is that traffic needs to be allowed both on **service A’s egress** and **service B’s ingress**.

## Next steps

From a machine with the cluster context configured, please share:

- The namespaces and pod names for A and B, plus the destination port and protocol.
- The output of `kubectl get networkpolicy -A -o yaml`, `kubectl get ciliumnetworkpolicy -A -o yaml`, and `kubectl get ciliumclusterwidenetworkpolicy -o yaml`.
- The output of `hubble observe --from-pod <namespace>/<pod-a> --to-pod <namespace>/<pod-b> --verdict DROPPED`.

That Hubble output should show whether Cilium is denying the flow and why. I haven’t changed any policies or cluster state.
````

### OLD 2

Local evidence: `/tmp/devops-regressions/pr7-final/S7/old-2/invocation.json`, `status.json`, `events.jsonl`.

````text
## Observations

I couldn’t inspect the live cluster from this workspace: `kubectl` has no current context, and it can’t connect to the default API address. I don’t have Hubble flow output or the policy manifests, so the cause isn’t confirmed.

## Likely cause

A policy selecting either endpoint may have put its ingress or egress direction into default-deny mode. For A to initiate a connection to B, A’s egress and B’s ingress must allow the flow. [Cilium policy enforcement docs](https://docs.cilium.io/en/stable/security/policy/intro/)

## Gather evidence

Run these from a machine with cluster access, substituting the pod names and namespaces:

```sh
hubble observe --from-pod <namespace-a>/<pod-a> \
  --to-pod <namespace-b>/<pod-b> --verdict DROPPED

kubectl get cnp,ccnp,networkpolicy -A
kubectl get pods -n <namespace-a> --show-labels
kubectl get pods -n <namespace-b> --show-labels
```

Hubble’s dropped flow should show whether Cilium reports `Policy denied`; its CLI is designed to inspect these flow verdicts. [Hubble CLI docs](https://docs.cilium.io/en/stable/observability/hubble/hubble-cli/)

If it shows a policy denial, check policies selecting both pods—including standard Kubernetes NetworkPolicies—and compare their selectors and port rules with the pods’ actual labels and the destination port. The fix will be a narrow allow rule at the direction that dropped, using the actual peer labels and port. Share the drop line, policy output, and service names/port, and I can identify the specific rule to adjust.
````

### OLD 3

Local evidence: `/tmp/devops-regressions/pr7-final/S7/old-3/invocation.json`, `status.json`, `events.jsonl`.

````text
## Observations

This workspace has no policy manifests. `kubectl` is installed, but no cluster context is configured, and the local `cilium` and `hubble` CLIs are unavailable, so I couldn’t inspect the live cluster.

## Evidence

`kubectl config current-context` reports `current-context is not set`; cluster queries then fail to connect to the Kubernetes API. I don’t have Hubble flow evidence, so I can’t confirm that Cilium is dropping the traffic.

## Root cause

Unconfirmed. Since the failure began after the policies were added, a missing or mismatched allow rule is a likely cause. Traffic may need to be allowed by both the source’s egress policy and the destination’s ingress policy; a plain Kubernetes `NetworkPolicy` can also affect the result.

## Fix

From a terminal with cluster access and Hubble available, watch the flow while reproducing the request:

```sh
hubble observe --from-pod <namespace>/<pod-a> --to-pod <namespace>/<pod-b> --verdict DROPPED -f
```

Also collect the policy and endpoint details:

```sh
kubectl get pods -n kube-system -l k8s-app=cilium -o wide
kubectl get cnp,ccnp,networkpolicy -A
kubectl get pods -n <namespace-a> --show-labels
kubectl get pods -n <namespace-b> --show-labels
kubectl get svc <service-b> -n <namespace-b> -o yaml
kubectl get endpointslices -n <namespace-b> -l kubernetes.io/service-name=<service-b> -o yaml
```

If Hubble reports `Policy denied`, the drop direction, pod labels, and destination port will identify which narrow peer-and-port allow rule is missing. If it reports no drop, the Service and EndpointSlice output can help distinguish a policy issue from a missing backend.

## Verification

After the policy is corrected, retry the request and check for forwarded flows:

```sh
hubble observe --from-pod <namespace>/<pod-a> --to-pod <namespace>/<pod-b> --verdict FORWARDED -f
```

Share the pod namespaces/names, service port and protocol, relevant policy YAML, and Hubble output, and I can narrow this to a specific rule.
````

### NEW 1

Local evidence: `/tmp/devops-regressions/pr7-final/S7/new-1/invocation.json`, `status.json`, `events.jsonl`.

````text
## Observations

This workspace has no service or policy manifests, and `kubectl config current-context` reports no active context. Cluster queries fell back to `localhost:8080` and were blocked, so I couldn’t inspect Cilium or Hubble directly.

## Root cause

I can’t identify the cause without a flow verdict and the policies selecting A and B. A policy may need to allow traffic in both directions: egress from A and ingress to B.

## Next checks

From a shell with cluster access, find B’s backend pods and capture a dropped flow:

```sh
kubectl get svc <service-b> -n <namespace> -o wide
kubectl get endpointslices -n <namespace> \
  -l kubernetes.io/service-name=<service-b> -o wide

hubble observe \
  --from-pod <namespace>/<service-a-client-pod> \
  --to-pod <namespace>/<service-b-backend-pod> \
  --verdict DROPPED --last 20

kubectl get cnp,ccnp,networkpolicy -A
```

The Hubble verdict should show which endpoint dropped the flow and why. Check the selected policies, pod labels, and destination port before changing anything. Share the flow output and the relevant policies, along with A and B’s namespaces, and I can help pinpoint the minimal fix. After applying a fix, verify that Hubble reports the flow as `FORWARDED`.
````

### NEW 2

Local evidence: `/tmp/devops-regressions/pr7-final/S7/new-2/invocation.json`, `status.json`, `events.jsonl`.

````text
## Observations

The workspace has no Cilium policy manifests. This environment has `kubectl`, but no current Kubernetes context, and it has no `cilium` or `hubble` CLI, so I can’t inspect the cluster from here.

## Evidence

The cluster query stopped at `kubectl config current-context`: **“current-context is not set.”** I don’t have a Hubble verdict or policy output to identify the drop.

## Root cause

Not confirmed yet. A flow needs to be allowed by **service A’s egress policy** and **service B’s ingress policy**. Cilium, Kubernetes NetworkPolicy, and Cilium clusterwide policies can all affect the same flow.

## Next step

From an environment with cluster access, run:

```sh
hubble observe --from-pod <namespace>/<service-a-pod> --to-pod <namespace>/<service-b-pod> --verdict DROPPED
kubectl get cnp,ccnp,netpol -A -o yaml
kubectl -n kube-system get pods -l k8s-app=cilium -o wide
```

Please share those outputs, plus whether A connects to B by service name/ClusterIP or directly by pod IP, and the destination port/protocol. That should show whether this is a policy denial, an unhealthy Cilium agent, or a service-path issue.
````

### NEW 3

Local evidence: `/tmp/devops-regressions/pr7-final/S7/new-3/invocation.json`, `status.json`, `events.jsonl`.

````text
## Observations

I can’t inspect the cluster from this workspace: `kubectl` is installed, but no current context is set, and the Cilium and Hubble CLIs aren’t available here. I haven’t found policy manifests in the workspace either.

## Evidence

I don’t have a Hubble flow or policy output yet, so the cause isn’t established. A common issue after adding policies is that traffic needs to be allowed by **both** source egress and destination ingress policies. Kubernetes NetworkPolicy, CiliumNetworkPolicy, and CiliumClusterwideNetworkPolicy can all affect the flow.

Please share the source and destination pod namespace/name, then run these from a shell with cluster access. Reproduce the request while observing the flow:

```sh
cilium status
hubble observe --from-pod <source-ns>/<source-pod> --to-pod <dest-ns>/<dest-pod> --last 20
hubble observe --verdict DROPPED --from-pod <source-ns>/<source-pod> --to-pod <dest-ns>/<dest-pod> --last 20
kubectl get netpol,ciliumnetworkpolicies.cilium.io,ciliumclusterwidenetworkpolicies.cilium.io -A -o yaml
```

The Hubble verdict and drop reason will help pinpoint whether this is a policy, endpoint, or service-path issue. I’d hold off on changing policies until we have that evidence.
````

## S8 — istio-debug

**Exact prompt:**

```text
We get intermittent 503s between two services since enabling Istio. Help.
```

| Arm | Replicate | Retry | Skill SHA-256 | Answer SHA-256 | Skill read | Earned | Content score |
|---|---:|---|---|---|---|---|---:|
| OLD | 1 | no | `5892aff64680d764041fe23b0c89720b377b7cc0d7a6a08bd19392fe7ee73090` | `22454c2b69229b369ccd8014cbb75dd686ce304180edae96b523a04512a83d41` | yes | C1 | 1/4 |
| OLD | 2 | no | `5892aff64680d764041fe23b0c89720b377b7cc0d7a6a08bd19392fe7ee73090` | `632b3dae50074f1873c7ec70493ef9ab0069762a14de2de93e11f2018837799b` | yes | C1 | 1/4 |
| OLD | 3 | no | `5892aff64680d764041fe23b0c89720b377b7cc0d7a6a08bd19392fe7ee73090` | `479dc647f49410689a153c51e9d14bfc65f864d255f35cc532571d9a84ff6db3` | yes | none | 0/4 |
| NEW | 1 | no | `ec990c9f78d3acb7c5ce1d296d234cdf448ace71e3164336f7237d3d6581560f` | `196e3643b01b2cd0263a106f7dab2caa72ebe62b566a6b1d5492461eff541799` | yes | C1 | 1/4 |
| NEW | 2 | no | `ec990c9f78d3acb7c5ce1d296d234cdf448ace71e3164336f7237d3d6581560f` | `44d12140bf557f0deb7da2122cb12badf9db0e68ec4c0f647ffdc9a9628612fe` | yes | C1 | 1/4 |
| NEW | 3 | no | `ec990c9f78d3acb7c5ce1d296d234cdf448ace71e3164336f7237d3d6581560f` | `8da7e5341d081482b27f05bba862d8b6822a440c616f3b74735ae682483b65e1` | yes | C1 | 1/4 |

Raw answer text follows. Invocation, status, and event traces remain under `/tmp/devops-regressions/pr7-final/` and `/tmp/devops-regressions/pr7-s2-retry/`.

### OLD 1

Local evidence: `/tmp/devops-regressions/pr7-final/S8/old-1/invocation.json`, `status.json`, `events.jsonl`.

````text
## Observations

I tried `istioctl analyze` first, but `istioctl` isn’t installed here. `kubectl` also can’t reach a cluster: it tried `localhost:8080` and the connection was denied. I can’t inspect live workloads or identify the failing hop from this workspace.

## Evidence and failing hop

I don’t have Envoy access logs yet, so the 503s can’t be classified. Please share the namespace and service names, plus:

```sh
istioctl analyze -n <namespace>
istioctl proxy-status
kubectl logs -n <namespace> <caller-pod> -c istio-proxy --since=15m | grep '503'
kubectl get endpoints <target-service> -n <namespace> -o yaml
```

If you can identify a destination pod, include its proxy log too:

```sh
kubectl logs -n <namespace> <destination-pod> -c istio-proxy --since=15m | grep '503'
```

If the log line includes them, keep the response flags and details, such as `UH`, `UF`, `UO`, `URX`, or `NR`. They point to different causes: no healthy endpoints, connection or mTLS failure, circuit-breaker overflow, exhausted retries, or no matching route.

## Root cause and fix

Not identified yet. Once we have the flags and endpoint state, I can trace the failing hop and suggest a targeted fix without changing live mesh config.
````

### OLD 2

Local evidence: `/tmp/devops-regressions/pr7-final/S8/old-2/invocation.json`, `status.json`, `events.jsonl`.

````text
## Observations

There are no service or Istio manifests in the workspace. `istioctl` is unavailable, and `kubectl` has no current context, so I couldn’t query the cluster or inspect Envoy logs. **I don’t have enough evidence to identify the failing hop or root cause yet.**

## Next evidence to collect

Run these read-only commands for the affected namespace and send me the output:

```sh
istioctl analyze -n <namespace>
istioctl proxy-status
kubectl logs <client-pod> -n <client-namespace> -c istio-proxy --since=30m
```

Please include a sidecar access log line for a 503, especially its response flag, response-code details, upstream cluster or host, and request ID if present. The flag helps narrow the cause: `UH` points to unavailable endpoints, `UF` to a connection failure, `NR` to missing routing, `UO`/`URX` to overflow or exhausted retries, and `UT` to a timeout.

Also send the two service names and namespaces. With that, I can trace the failing hop and recommend a specific fix.
````

### OLD 3

Local evidence: `/tmp/devops-regressions/pr7-final/S8/old-3/invocation.json`, `status.json`, `events.jsonl`.

````text
## Observations

`istioctl` is unavailable here, so I couldn’t run `istioctl analyze`. There’s also no Kubernetes context configured, and this environment can’t reach the cluster API. I couldn’t inspect the pods, logs, or Istio resources, so I can’t identify the failing hop yet.

## Evidence and failing hop

There’s no Envoy response flag or proxy status to classify the 503s. The failing hop is still unknown.

From a shell with cluster access, collect the caller sidecar’s 503s and check whether the destination has endpoints:

```sh
SRC_NS=<caller-namespace>
SRC_POD=<caller-pod>
DST_NS=<destination-namespace>
DST_SVC=<destination-service>

kubectl logs -n "$SRC_NS" "$SRC_POD" -c istio-proxy --since=15m \
  | rg '503|UH|UF|NR|UO|URX|UT|LR|UC' | tail -80

kubectl get endpointslice -n "$DST_NS" \
  -l "kubernetes.io/service-name=$DST_SVC" -o wide

istioctl analyze -n "$DST_NS"
istioctl proxy-status
```

If you can reproduce the issue, watch the destination endpoints while doing so:

```sh
kubectl get endpointslice -n "$DST_NS" \
  -l "kubernetes.io/service-name=$DST_SVC" -w
```

## Root cause and fix

Undetermined until we have the response flags and endpoint state. The flags will narrow it down: `UH` points to no healthy upstream endpoints; `UF` to a connection failure such as an mTLS mismatch or an app port/listener problem; `NR` to a missing route; and `UO` or `URX` to connection-pool or retry limits. If 503s coincide with deploys, check endpoint readiness and draining.

I’d hold off on changing mesh config until that evidence identifies the failing path. Share the relevant log lines, namespace/service names, and `proxy-status` output, and I can trace the cause and suggest a focused fix.
````

### NEW 1

Local evidence: `/tmp/devops-regressions/pr7-final/S8/new-1/invocation.json`, `status.json`, `events.jsonl`.

````text
## Observations

The workspace contains no service or Istio configuration. `istioctl` is unavailable, and `kubectl` has no current context; attempts to query the cluster failed. I couldn’t run `istioctl analyze` or inspect live proxy status, logs, or endpoints.

## Evidence

There’s no Envoy response flag or access-log line yet, so the failing hop and root cause are unknown. For intermittent 503s, that flag is the best starting point.

## Next checks

From a machine with cluster access, please collect:

```sh
istioctl analyze -n <namespace>
istioctl proxy-status
kubectl get pods -n <namespace> -o wide
kubectl logs -n <namespace> <caller-pod> -c istio-proxy --since=15m
```

Also check the destination’s endpoints and the relevant `VirtualService`, `DestinationRule`, and `PeerAuthentication` resources. Share one failing proxy log line with its response flags, along with the caller and destination service names and namespaces. Redact sensitive values.

Common flags narrow the investigation: `UH` points to missing or unready endpoints; `UF` to a connection failure such as an mTLS mismatch or an app port issue; `NR` to routing; and `UO` or `URX` to circuit-breaking or exhausted retries. These are leads, not a diagnosis without the log evidence.

## Failing hop, root cause, and fix

Not determined yet. I haven’t changed any mesh configuration. Once the log and config evidence identify the cause, I can recommend a specific fix and a read-only verification command.
````

### NEW 2

Local evidence: `/tmp/devops-regressions/pr7-final/S8/new-2/invocation.json`, `status.json`, `events.jsonl`.

````text
## Observations

I can’t inspect the live mesh from this workspace: there are no service manifests here, `kubectl` has no current context, and `istioctl` isn’t installed. So I don’t have proxy evidence to identify the failing hop yet.

## Evidence to collect

During a failure window, check the caller and destination sidecars for a 503 and its Envoy response flag. If access logs are enabled, include the response flag, response code details, upstream host, and request ID.

Run these read-only checks from a machine with cluster access:

```sh
istioctl analyze -n <namespace>
istioctl proxy-status

kubectl logs -n <caller-ns> <caller-pod> -c istio-proxy --since=15m
kubectl logs -n <dest-ns> <dest-pod> -c istio-proxy --since=15m

kubectl get endpointslice -n <dest-ns> \
  -l kubernetes.io/service-name=<service> -o wide
```

If you can, also get the caller’s view of the service endpoints:

```sh
istioctl proxy-config endpoints <caller-pod> -n <caller-ns> \
  --cluster "outbound|<port>||<service>.<dest-ns>.svc.cluster.local"
```

## How to interpret the flag

- `UH`: no healthy upstream; check readiness, endpoint churn, and DestinationRule subset labels.
- `UF`: upstream connection failed; check mTLS on both sides and whether the app is listening on the expected port and address.
- `UO` or `URX`: connection limit or retry exhaustion; inspect DestinationRule limits and the underlying attempt failures.
- `UT`: upstream timed out; compare service latency with the route timeout.
- `NR`: no matching route; check the request host and VirtualService rules.

I’d check `proxy-status` before changing routing rules, and avoid changing mesh config until the 503s are classified. Share the relevant sidecar log lines, `proxy-status`, and endpoint output, along with the caller and destination namespaces, and I can narrow down the cause.
````

### NEW 3

Local evidence: `/tmp/devops-regressions/pr7-final/S8/new-3/invocation.json`, `status.json`, `events.jsonl`.

````text
## Observations

The workspace has no service manifests, and I couldn’t inspect the cluster: `kubectl` has no current context, and the API request to its default address was blocked. `istioctl` isn’t installed here, so I couldn’t run `istioctl analyze` or check proxy sync.

## Evidence and failing hop

I don’t have proxy logs or Envoy response flags yet, so I can’t identify which hop is failing or name a root cause. The caller’s `istio-proxy` log is the best place to start: its response flag usually narrows down the cause.

Run these where you have cluster access, substituting the namespaces and pod names:

```bash
istioctl analyze -n <source-namespace>
istioctl proxy-status

kubectl logs -n <source-namespace> <caller-pod> -c istio-proxy --since=15m \
  | rg '503|UH|UF|UO|URX|NR|UT|UC|LR'

kubectl logs -n <destination-namespace> <server-pod> -c istio-proxy --since=15m \
  | rg '503|UH|UF|UO|URX|NR|UT|UC|LR'
```

The flags point to different lines of investigation:

- `UH`: no healthy upstream endpoints; check readiness, pod churn, and any DestinationRule subset labels.
- `UF`: upstream connection failed; check mTLS settings, the app’s listen address, and the service port.
- `NR`: no route matched; check the request host and VirtualService rules.
- `UO` or `URX`: connection pool overflow or retries exhausted; inspect the DestinationRule and the underlying attempt failures.
- `UT`: upstream timeout; compare the route timeout with service latency.

Send the affected service pair and namespace, plus the failing caller-side log line and `istioctl proxy-status` output. Redact sensitive values, but keep the response flag.
````
