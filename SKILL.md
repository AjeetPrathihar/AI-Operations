---
name: k8s-pod-troubleshooter
description: Kubernetes troubleshooting skill for diagnosing Pending, CrashLoopBackOff, ImagePullBackOff, CreateContainerConfigError, PVC, Service, DNS, NetworkPolicy, probe, and scheduling issues using read-only kubectl diagnostics.
version: 1.0.0
author: Ajeet Singh Parihar
---

# Kubernetes Pod Troubleshooter

Sample prompts:
- "why is pod payments-api-7d9f stuck in Pending in namespace shop?"
- "diagnose pods in namespace shop on cluster scus-prod-a6"
- "my service has no endpoints, check why"
- "pod keeps failing readiness probe, help me find out why"
- "check the PVC for pod db-0"

## Ground rules

1. **Read-only by default.** Use only `get`, `describe`, `logs`, `top`, `events`, and `auth can-i`. Use `kubectl exec` (nslookup, wget, cat) only when RBAC permits and the user has approved diagnostic shell execution; some organizations treat exec as interactive access. Do not `delete`, `patch`, `apply`, `scale`, `rollout restart`, `cordon`, or `drain` unless the user explicitly asks. Even then, show the exact command and ask for confirmation first (use `--dry-run=server` where supported).
2. **Never print Secret values.** Check that a Secret and its keys exist (`kubectl get secret NAME -n NS -o json | jq -r '.data | keys[]'`). Do not decode or echo values.
3. **Always pin context and namespace.** Confirm with `kubectl config current-context` and `kubectl config view --minify` (cluster, user, default namespace) before the first check, and pass `-n NAMESPACE` on every command. Engineers often troubleshoot the wrong cluster; state the context back to the user. If the user gave no namespace or cluster, ask once. Do not guess for production.
4. **Evidence before conclusions.** State the root cause only when an event message, log line, or field value supports it. If evidence is ambiguous, say so and list the next check.
5. **Stop on access errors.** If a command returns Forbidden or a connection error, report it and say which permission or VPN/context is needed. Do not work around it.
6. **Report every blocker, not just the first.** When evidence points to multiple causes, identify the primary blocker and the secondary blockers separately. Do not stop after finding the first issue. Pending pods often have Insufficient CPU AND an untolerated taint AND a topology spread violation at once.

## Decision order

Work top to bottom and do not skip ahead:

1. Does the pod exist? (If not, check the owner: ReplicaSet/StatefulSet/Job events, quota, admission.)
2. Which phase is it in?
3. Is it scheduled? (Step 2)
4. Did the containers start? (Step 3 and Step 4)
5. Is it Ready? (Step 5a)
6. Is it reachable? (Step 5b and 5c)

## Step 1: Triage the pod phase

```bash
kubectl get pod POD -n NS -o wide
kubectl get pod POD -n NS -o jsonpath='{.status.phase}{"\n"}{.status.conditions[*].type}={.status.conditions[*].status}{"\n"}'
kubectl get events -n NS --field-selector involvedObject.name=POD --sort-by=.lastTimestamp
```

| Observation | Go to |
|---|---|
| Phase `Pending`, no node assigned (`NODE` is `<none>`) | Step 2 (scheduling) |
| Phase `Pending`, node assigned, `ContainerCreating` or `PodInitializing` | Step 3 (volumes, config, image) |
| Phase `Running` but `READY 0/1` | Step 5 (probes, endpoints) |
| `CrashLoopBackOff`, `Error`, `OOMKilled` | Step 4 (logs) |
| Pod doesn't exist at all | Check the owner: `kubectl describe rs/deploy/sts -n NS`, look for `exceeded quota` or admission errors |

## Step 2: Pod Pending and unscheduled

```bash
kubectl describe pod POD -n NS
```

Read the `Events:` section, specifically the `FailedScheduling` message, e.g. `0/12 nodes are available: 3 Insufficient cpu, 4 node(s) had untolerated taint ..., 5 node(s) didn't match Pod's node affinity/selector.` Each count is a reason a node was rejected. The final blocker is any reason that remains true across all candidate nodes: for example, 9 nodes with Insufficient cpu and 1 node with an untolerated taint still cannot schedule the pod if the only node with free CPU is the tainted one. Do not assume the largest count is the blocker. Match every message to a cause below.

| Event message contains | Cause | Verify with |
|---|---|---|
| `Insufficient cpu` / `Insufficient memory` | Requests exceed free capacity on every node | `kubectl get pod POD -n NS -o jsonpath='{.spec.containers[*].resources}'` and `kubectl describe nodes \| grep -A8 "Allocated resources"`; `kubectl top nodes` |
| `unbound immediate PersistentVolumeClaims` / `persistentvolumeclaim ... not found` | PVC not bound or missing | Step 3a |
| `didn't match Pod's node affinity/selector` | `nodeSelector` or `nodeAffinity` matches no node | `kubectl get pod POD -n NS -o jsonpath='{.spec.nodeSelector}{"\n"}{.spec.affinity.nodeAffinity}'` vs `kubectl get nodes --show-labels` (or `-L key`) |
| `had untolerated taint {key: value}` | Node taint without matching toleration | `kubectl get nodes -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints`; compare with `kubectl get pod POD -n NS -o jsonpath='{.spec.tolerations}'` |
| `didn't match pod topology spread constraints` | `maxSkew` / `whenUnsatisfiable: DoNotSchedule` can't be met | `kubectl get pod POD -n NS -o jsonpath='{.spec.topologySpreadConstraints}'`; count existing pods per zone/node with `kubectl get pods -n NS -l LABEL -o wide` |
| `didn't match pod anti-affinity rules` | `requiredDuringScheduling` anti-affinity too strict (e.g. one pod per node with fewer nodes than replicas) | `kubectl get pod POD -n NS -o jsonpath='{.spec.affinity.podAntiAffinity}'`; compare replica count to eligible nodes |
| `node(s) were unschedulable`, `not ready`, or `no nodes available to schedule pods` | Cordoned or NotReady nodes, or no nodes in the pool | `kubectl get nodes`; `kubectl describe node NODE \| grep -A10 Conditions` |
| Pod missing; ReplicaSet event `exceeded quota` | ResourceQuota exhausted | `kubectl describe resourcequota -n NS`; `kubectl describe rs -n NS` |
| `Insufficient` for `ephemeral-storage`, GPU, or extended resource | Requests for a scarce resource | Check `resources.requests` against node `Allocatable` |
| `RuntimeClass ... not found` / `no runtime for ...` | `runtimeClassName` missing or not supported on any node | `kubectl get pod POD -n NS -o jsonpath='{.spec.runtimeClassName}'`; `kubectl get runtimeclass`; check its `scheduling.nodeSelector` and tolerations |

Notes:
- Requests (not actual usage) drive scheduling. A node can look idle in `kubectl top` and still be full by requests.
- Also check `priorityClassName`; preemption may be involved.
- Suggest fixes as options (lower requests, add or scale nodes, fix selector/label, add toleration, relax constraint to `preferred...`/`ScheduleAnyway`, raise quota) and do not apply them.

## Step 3: Assigned to a node but not starting

### 3a. PVC

```bash
kubectl get pvc -n NS
kubectl describe pvc PVC -n NS            # status, StorageClass, events
kubectl get storageclass                  # exists? volumeBindingMode?
kubectl get pv | grep PVC
```
Look for: `Pending` PVC, a nonexistent StorageClass, `WaitForFirstConsumer` (normal until the pod schedules), zone mismatch between PV and node, access mode conflict (`ReadWriteOnce` already attached elsewhere, shown as `Multi-Attach error`), or provisioner errors in the PVC events.

### 3b. ConfigMap and Secret references

```bash
kubectl get pod POD -n NS -o jsonpath='{range .spec.volumes[*]}{.name}{"\t"}{.configMap.name}{.secret.secretName}{"\n"}{end}'
kubectl get pod POD -n NS -o yaml | grep -E "configMapKeyRef|secretKeyRef|configMapRef|secretRef" -A3
kubectl get configmap CM -n NS
kubectl get secret SECRET -n NS -o json | jq -r '.data | keys[]'   # keys only, never values
```
Typical events: `configmap "x" not found`, `secret "x" not found`, `CreateContainerConfigError` with `couldn't find key K in ConfigMap/Secret`. Check the name, namespace, and exact key spelling. A reference marked `optional: true` will not block startup.

### 3c. Image and registry

`ErrImagePull` / `ImagePullBackOff` in events: verify image name and tag, `imagePullSecrets` exist in the namespace (`kubectl get secret -n NS`), and registry reachability.

### 3d. Admission, PodSecurity, LimitRange, RuntimeClass

Pods can be rejected or altered before they ever start. Check the owner's events (ReplicaSet, StatefulSet, Job) when no pod appears, and `CreateContainerConfigError` / `Forbidden` events when it does.

**PodSecurity Admission.** Typical messages: `violates PodSecurity "restricted:latest"`, `runAsNonRoot` / `runAsRoot forbidden`, `hostPath volumes forbidden`, `privileged container forbidden`, `allowPrivilegeEscalation != false`, `unrestricted capabilities`.
```bash
kubectl get ns NS --show-labels | tr ',' '\n' | grep pod-security
kubectl get pod POD -n NS -o jsonpath='{.spec.securityContext}{"\n"}{.spec.containers[*].securityContext}'
kubectl describe rs -n NS | grep -i -B1 -A3 "forbidden\|violates"
```
Compare the namespace labels `pod-security.kubernetes.io/enforce|warn|audit` (privileged, baseline, restricted) with the pod's `securityContext` (`runAsNonRoot`, `privileged`, `capabilities`, `seccompProfile`, `hostPath`, `hostNetwork`). Suggest fixing the pod spec; never suggest weakening the namespace label without the owner's approval.

**LimitRange defaulting.** A LimitRange can inject default requests/limits the user never wrote, or reject a pod outside its min/max, which explains surprise OOMKills, CPU throttling, or `Insufficient` scheduling failures.
```bash
kubectl describe limitrange -n NS
kubectl get pod POD -n NS -o jsonpath='{.spec.containers[*].resources}'
```
If the pod's resources are non-empty but the manifest set none, the LimitRange injected them. Look for `default`, `defaultRequest`, `min`, `max`, and `maxLimitRequestRatio`; events like `maximum memory usage per Container is ...` mean rejection.

**RuntimeClass.** `kubectl get runtimeclass` and compare with `.spec.runtimeClassName` on the pod. A missing class blocks pod creation; a class with a restrictive `scheduling.nodeSelector` can leave the pod Pending (see Step 2). Also check admission webhooks that mutate pods: `kubectl get mutatingwebhookconfigurations,validatingwebhookconfigurations` and the event text naming the webhook.

## Step 4: Crashing containers (logs)

```bash
kubectl logs POD -n NS -c CONTAINER                 # current
kubectl logs POD -n NS -c CONTAINER --previous      # last crashed instance
kubectl logs POD -n NS --all-containers --tail=200
kubectl get pod POD -n NS -o jsonpath='{.status.containerStatuses[*].lastState}'
```
Read `lastState.terminated` for `reason` and `exitCode`:

| Exit code | Meaning | Next step |
|---|---|---|
| 1 | Application error | Read logs (`--previous`) |
| 126 | Command found but not executable | Check file permissions, entrypoint, `command`/`args` |
| 127 | Command not found | Check image contents, `command`/`args`, PATH |
| 134 | SIGABRT: application panic or abort | Read logs for the panic/assert/stack trace |
| 137 | SIGKILL: `OOMKilled` (reason) or killed by kubelet/liveness | Raise memory limit or fix the leak; if reason isn't OOMKilled, check liveness probe and node pressure |
| 139 | SIGSEGV: segmentation fault | Native crash; suspect image/arch mismatch, bad library, or an app bug; capture logs and image version |
| 143 | SIGTERM: container terminated gracefully | Normal on rollout, eviction, or scale-down; if looping, check liveness probe and who is sending the signal |

Check init containers too: `kubectl logs POD -n NS -c INIT_NAME`.

Resource pressure (supports OOM and CPU throttling analysis; needs metrics-server):
```bash
kubectl top pod POD -n NS --containers
kubectl top node
kubectl describe node NODE | grep -A8 "Allocated resources"
```
Compare usage with `resources.limits`; usage near the memory limit points to OOM, CPU at the limit points to throttling, node `MemoryPressure`/`DiskPressure` conditions point to eviction.

## Step 5: Running but not Ready or unreachable

### 5a. Probes

```bash
kubectl get pod POD -n NS -o jsonpath='{.spec.containers[*].readinessProbe}{"\n"}{.spec.containers[*].livenessProbe}{"\n"}{.spec.containers[*].startupProbe}'
kubectl describe pod POD -n NS | grep -iE "probe failed|unhealthy|killing"
```
Common faults: wrong path or port; `initialDelaySeconds` too short for slow startup (suggest a `startupProbe`); `timeoutSeconds` too low; liveness probe depending on an external service, which causes restart loops; probe port differs from the container port. Test the endpoint from inside the pod:
```bash
kubectl exec POD -n NS -c CONTAINER -- wget -qO- http://localhost:PORT/PATH
```

### 5b. Service, Endpoints, EndpointSlices

```bash
kubectl get svc SVC -n NS -o wide
kubectl get endpoints SVC -n NS
kubectl get endpointslices -n NS -l kubernetes.io/service-name=SVC
kubectl get pods -n NS --show-labels
kubectl get svc SVC -n NS -o jsonpath='{.spec.selector}'
```
Empty endpoints usually means (1) the Service selector doesn't match pod labels, (2) pods aren't Ready (probes fail, so they're listed under `notReadyAddresses`), or (3) `targetPort` doesn't match the container port. Compare selector vs labels character by character.

**Verify the port chain.** Many outages are a port mismatch: Service `port` = 80, `targetPort` = 8080, but the container listens on 9090.
```bash
kubectl get svc SVC -n NS -o yaml          # .spec.ports[].port / targetPort / name
kubectl get pod POD -n NS -o yaml          # .spec.containers[].ports[].containerPort and probe ports
```
`targetPort` must equal the port the process actually listens on (check app logs or `kubectl exec ... -- cat /proc/net/tcp` if allowed), and a named `targetPort` must match a `containerPort` name.

### 5c. DNS and connectivity from the pod

```bash
kubectl exec -it POD -n NS -c CONTAINER -- nslookup SVC.NS.svc.cluster.local
kubectl exec -it POD -n NS -c CONTAINER -- cat /etc/resolv.conf
```
If the image has no `nslookup`, use an ephemeral debug container:
```bash
kubectl debug -it POD -n NS --image=busybox:1.36 --target=CONTAINER -- nslookup kubernetes.default
```
If cluster DNS fails, check CoreDNS: `kubectl get pods -n kube-system -l k8s-app=kube-dns` and its logs. If DNS works but connections fail, test with `nc -zv HOST PORT` and check NetworkPolicies:
```bash
kubectl get networkpolicy -n NS
kubectl describe networkpolicy -n NS
kubectl get ns --show-labels               # for namespaceSelector matches
```
Look for: ingress denied (no rule allows the source), egress denied (including DNS to kube-system on port 53), `namespaceSelector` mismatch (namespace lacks the expected label), `podSelector` mismatch (labels differ from the policy), and a default-deny policy with no allow rules for this pod.

## Output format

Reply with these sections, concise:

1. **Verdict:** one sentence naming the primary root cause (or "inconclusive"), followed by any secondary blockers.
2. **Impact:** what the user sees because of this issue (e.g. "Pod cannot be scheduled. Service has no endpoints.").
3. **Evidence:** the exact event line, log line, or field value that proves it (quote it), plus the command that produced it.
4. **Fix options:** 1 to 3 concrete changes, with the exact manifest field or command. Mark each as "not applied".
5. **Also noticed:** other issues seen along the way (e.g. missing probes, requests with no limits). Omit if none.

If several causes apply (for example Insufficient memory on some nodes and a taint on others), list every cause with its node count and say which would still block scheduling after the first is fixed.
