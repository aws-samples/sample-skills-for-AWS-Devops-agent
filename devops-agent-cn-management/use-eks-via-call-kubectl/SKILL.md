---
name: use-eks-via-call-kubectl
description: Use when any task involves inspecting EKS pod status, pod events,
  deployment state, or container logs on the China-region cluster bjs-web
  (cn-north-1, account aws-cn-2). Triggers on phrases like "kubectl get pods",
  "pod not ready", "ImagePullBackOff", "CrashLoopBackOff", "pod pending",
  "pod status", "check deployment", "pod logs", "container logs", "查pod",
  "pod状态", or any task description involving kubectl commands against bjs-web.
  CRITICAL: use_kubectl does NOT work for China-region EKS. Always use the
  call_kubectl tool from the aws-cn-2 MCP server instead.
---

# EKS Inspection via call_kubectl (China Region)

## Why `use_kubectl` fails here

`use_kubectl` is a platform-level tool with a hardcoded account allowlist
limited to the global (`aws`) partition. The cluster `bjs-web` lives in
account `107422471498`, `cn-north-1` (`aws-cn` partition). Token-based EKS
authentication cannot cross partition boundaries.

**Do not retry `use_kubectl` with any account ID. It will never work.**

## The correct tool: `call_kubectl`

The `aws-cn-2` MCP server exposes a `call_kubectl` tool alongside `call_aws`.
Same connection, same endpoint — no additional MCP server needed.

Allowed verbs: `get`, `describe`, `logs`, `top`, `explain`,
`version`, `cluster-info`, `api-resources`, `api-versions`

Write verbs (`apply`, `delete`, `patch`, `exec`, etc.) are blocked by the
tool itself — output them as a draft command for human approval instead.

## Common substitutions

| Original intent | Use this instead |
|---|---|
| `kubectl get pods -n bjs-web -o wide` | `call_kubectl("kubectl get pods -n bjs-web -o wide")` |
| `kubectl describe pod <pod> -n bjs-web` | `call_kubectl("kubectl describe pod <pod> -n bjs-web")` |
| `kubectl logs <pod> -n bjs-web --since=1h` | `call_kubectl("kubectl logs <pod> -n bjs-web --since=1h")` |
| `kubectl get events -n bjs-web --sort-by=.lastTimestamp` | `call_kubectl("kubectl get events -n bjs-web --sort-by=.lastTimestamp")` |
| `kubectl get deployments -n bjs-web` | `call_kubectl("kubectl get deployments -n bjs-web")` |

## Standard incident investigation sequence

For a pod-not-ready alarm on bjs-web:

```
# 1. Pod overview
call_kubectl("kubectl get pods -n bjs-web -o wide")

# 2. For any non-Running pod, describe it
call_kubectl("kubectl describe pod <pod-name> -n bjs-web")

# 3. Events (sorted by time)
call_kubectl("kubectl get events -n bjs-web --sort-by=.lastTimestamp")

# 4. Deployment status
call_kubectl("kubectl get deployments -n bjs-web")

# 5. Logs if pod is running or was recently running
call_kubectl("kubectl logs <pod-name> -n bjs-web --since=30m")
```

## Failure pattern quick reference

| What you see in describe/events | Root cause |
|---|---|
| `ImagePullBackOff` / `ErrImagePull` | Image tag does not exist in ECR — fault L6 |
| `OOMKilled` | Memory limit hit — fault L9 neighbourhood |
| `CrashLoopBackOff` | Container exits — check logs next |
| `Insufficient cpu` / `Insufficient memory` | Node resource exhausted |
| `FailedScheduling` | No schedulable node |

## Things not to do

- **Do not** call `use_kubectl` for China-region EKS. Ever.
- **Do not** add a new MCP server for kubectl access — `call_kubectl` already
  lives inside the existing `aws-cn-2` connection.
- **Do not** use `call_aws` + CloudWatch as a kubectl substitute — `call_kubectl`
  gives direct Kubernetes API access and is more accurate.
- **Do not** issue write verbs via `call_kubectl` — they are blocked. Draft
  the remediation command and request human approval via the mitigation skill.
