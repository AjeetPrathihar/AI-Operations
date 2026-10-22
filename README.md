# k8s-pod-troubleshooter

Diagnoses Kubernetes pods that are Pending, not Ready, crashing, blocked by admission, or unreachable. It uses read-only `kubectl` checks only and reports the root cause with suggested fixes. It never applies them.

## What it covers

- **Pending / scheduling:** insufficient CPU/memory, unbound PVC, nodeSelector/affinity, taints, topology spread, pod anti-affinity, NotReady nodes, ResourceQuota, RuntimeClass
- **Not starting:** PVC problems, missing ConfigMap/Secret references, image pull errors
- **Admission:** PodSecurity violations, LimitRange defaulting, RuntimeClass, mutating/validating webhooks
- **Crashing:** logs, exit codes 1, 126, 127, 134, 137, 139, 143, OOMKilled, resource pressure
- **Not Ready / unreachable:** probes, Service/Endpoints/EndpointSlices, port mismatches, DNS, NetworkPolicies

## Requirements

- `kubectl` on PATH, with access to the target cluster (VPN if required)
- `jq` for Secret key listing
- RBAC permission for `get`, `describe`, `logs` and, where used, `exec`

## Safety

- Read-only by default. No delete, patch, apply, scale, restart, cordon or drain without explicit user request and confirmation.
- Secret values are never printed, only key names.
- `kubectl exec` is used only when RBAC permits and the user has approved it.
- The skill confirms the cluster context and namespace before the first check.

## Security and compliance

- **Least privilege:** only read verbs (`get`, `describe`, `logs`, `top`, `auth can-i`) run by default. The skill works with a read-only RBAC role and needs no write access.
- **No secret exposure:** Secret values are never decoded, printed, or logged, which keeps credentials out of chat history and transcripts. Only key names are checked.
- **Change control:** the skill makes no changes. Fixes are suggested and marked "not applied", so every modification still goes through your normal change process (pull request, pipeline, approval). This avoids untracked manual changes and configuration drift.
- **Production safety:** the skill confirms context and namespace first and stops on Forbidden or connection errors instead of working around them. This reduces the risk of troubleshooting the wrong cluster or bypassing access controls.
- **Controlled access:** `kubectl exec` needs both RBAC permission and user approval, since some organizations treat it as interactive access.
- **Policy-aware diagnosis:** it reads PodSecurity, LimitRange and admission rejections as guardrails. It recommends fixing the workload spec and never weakening namespace policy without the owner's approval.
- **Auditability:** every diagnosis quotes the evidence (event line, log line, or field value) and the command that produced it. Findings can be reviewed or attached to an incident record.
- **Data minimization:** it reads only what the diagnosis needs, namespace-scoped, and does not collect cluster-wide data unless a check requires it, such as node capacity for scheduling.

## Example prompts

- "why is pod payments-api-7d9f stuck in Pending in namespace shop?"
- "diagnose pods in namespace shop on cluster scus-prod-a6"
- "my service has no endpoints, check why"
- "pod keeps failing readiness probe, help me find out why"
- "check the PVC for pod db-0"
- "pod exits with code 139, what does it mean?"

## Output

Each diagnosis has: **Verdict**, **Impact**, **Evidence**, **Fix options** (marked "not applied"), and **Also noticed**.

## Files

```
k8s-pod-troubleshooter/
  SKILL.md    skill definition and workflow
  README.md   this file
```
