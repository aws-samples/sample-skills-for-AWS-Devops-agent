---
name: eks-resilience-checker
description: Assess Amazon EKS cluster resilience against 26 best-practice checks across application workloads (A1-A14), control plane (C1-C5), and data plane (D1-D7), and produce a structured assessment.json that can drive chaos experiments. Use this skill when the user asks to evaluate EKS cluster resilience, run a resilience assessment, audit EKS best practices, identify singleton pods, missing PDBs, missing HPA, multi-AZ spread issues, control plane logging gaps, endpoint access misconfigurations, or prepare an EKS workload for chaos engineering. Triggers on phrases like "EKS resilience", "EKS readiness", "Kubernetes resilience check", "cluster assessment", "EKS best practices audit", "韧性评估", "集群评估", "resiliency check".
---

# EKS Resilience Checker

## Role Definition

You are a senior AWS EKS resilience assessment expert. You perform 26 automated checks across 3 categories — Application Workloads (A1-A14), Control Plane (C1-C5), and Data Plane (D1-D7) — against an Amazon EKS cluster. You output structured assessment results that can drive chaos experiments via `chaos-engineering-on-aws`.

## Prerequisites

### Required Tools

| Tool | Purpose | Verify |
|------|---------|--------|
| `kubectl` | K8s API queries | `kubectl version --client` |
| `aws` CLI | EKS describe-cluster, addon queries | `aws sts get-caller-identity` |
| `jq` | JSON parsing | `jq --version` |

### EKS Authentication

Two methods available: **IAM kubeconfig** (recommended for interactive use) or **Static Service Account Token** (for CI/CD).
See [eks-auth-setup.md](references/eks-auth-setup.md) for detailed setup instructions.

### Required Permissions

| Scope | Permissions |
|-------|------------|
| Kubernetes RBAC | `get`, `list` on: pods, deployments, statefulsets, daemonsets, services, nodes, pdb, hpa, vpa, webhooks, resourcequotas, limitranges, configmaps, namespaces, crds |
| AWS IAM | `eks:DescribeCluster`, `eks:ListAddons`, `eks:DescribeAddon`, `eks:ListAccessEntries` |

### Optional MCP Server

| Server | Package | Purpose |
|--------|---------|---------|
| eks-mcp-server | `awslabs.eks-mcp-server` | K8s resource queries (alternative to kubectl) |

When MCP is unavailable, fall back to `kubectl` + `aws` CLI direct calls.

## State Persistence

All output goes to `output/` directory:

```
output/
├── step1-cluster.json          # Cluster discovery results
├── assessment.json             # Structured results (26 checks) — chaos skill input
├── assessment-report.md        # Human-readable Markdown report
├── assessment-report.html      # HTML report (inline CSS, color-coded)
└── remediation-commands.sh     # Fix script (requires manual execution)
```

On startup, check for existing `output/`. If prior results exist, ask: **continue from last run** or **start fresh**.

---

## Four-Step Workflow

### Step 1: Cluster Discovery

1. **Get cluster name** — User provides it, or auto-detect:
   ```bash
   kubectl config current-context | sed 's|.*:cluster/||'
   ```

2. **Describe cluster** — Collect metadata:
   ```bash
   aws eks describe-cluster --name {CLUSTER_NAME} --region {REGION} --output json
   ```
   Extract: `kubernetesVersion`, `platformVersion`, `vpcId`, endpoint config, logging, tags, addons.

3. **List addons**:
   ```bash
   aws eks list-addons --cluster-name {CLUSTER_NAME} --region {REGION} --output json
   ```

4. **Determine target namespaces** — List non-system namespaces, present to user for confirmation:
   ```bash
   kubectl get namespaces -o json | jq -r '[.items[].metadata.name | select(test("^kube-") | not) | select(. != "kube-system" and . != "kube-public" and . != "kube-node-lease")]'
   ```

5. **Detect EKS Auto Mode**:
   ```bash
   aws eks describe-cluster --name {CLUSTER_NAME} --query 'cluster.computeConfig.enabled' --output text
   ```
   If `true`, flag for D7 auto-pass and adjust node-related checks.

6. **Detect Fargate profiles**:
   ```bash
   aws eks list-fargate-profiles --cluster-name {CLUSTER_NAME} --output json
   ```
   If Fargate profiles exist, skip inapplicable checks (A3, D1) for Fargate workloads.

**Output**: Save to `output/step1-cluster.json`. Confirm cluster, region, namespaces with user.

---

### Step 2: Automated Checks (26 Items)

Run all 26 checks against the confirmed cluster and namespaces.

**Commands and PASS/FAIL criteria**: See [check-commands.md](references/check-commands.md) for exact kubectl/aws CLI commands for each check.
**MCP alternative**: See [eks-resiliency-checks-mcp.md](references/eks-resiliency-checks-mcp.md) for MCP-based execution.
**Check descriptions and rationale**: See [EKS-Resiliency-Checkpoints.md](references/EKS-Resiliency-Checkpoints.md).

**Namespace filter** (used throughout):
```bash
TARGET_NS="namespace1,namespace2,..."  # from Step 1
```

For namespace-scoped checks, loop over `TARGET_NS`. For cluster-wide checks (A7, C1-C5, D1-D2, D6-D7), no filter needed.

**26 Checks Overview**:

| Category | Checks | Severity Mix |
|----------|--------|-------------|
| **Application (A1-A14)** | Singleton pods, replicas, anti-affinity, probes, PDB, metrics server, HPA, VPA, preStop hooks, service mesh, monitoring, logging | 4 Critical, 7 Warning, 3 Info |
| **Control Plane (C1-C5)** | Control plane logs, authentication, large cluster optimization, endpoint access, webhook catch-all | 1 Critical, 4 Warning |
| **Data Plane (D1-D7)** | Node autoscaler, multi-AZ spread, resource requests/limits, ResourceQuotas, LimitRanges, CoreDNS metrics, CoreDNS addon | 3 Critical, 3 Warning, 1 Info |

**Check Result Format** (per check):
```json
{
  "id": "A1",
  "name": "Avoid Running Singleton Pods",
  "category": "application",
  "severity": "critical",
  "status": "PASS",
  "findings": [],
  "resources_affected": [],
  "remediation": "",
  "chaos_experiment_recommendation": null
}
```

For FAIL results, populate `findings` with descriptions, `resources_affected` with `namespace/resource-name`, `remediation` with fix commands (see [remediation-templates.md](references/remediation-templates.md)).

---

### Step 3: Generate Reports

After all 26 checks, generate four output files to `output/`:

#### 3.1 assessment.json
Structured results with schema:
- `cluster_name`, `region`, `kubernetes_version`, `platform_version`, `timestamp`, `target_namespaces`
- `summary`: total_checks, passed, failed, info, critical_failures, compliance_score
- `checks[]`: array of check results from Step 2
- `experiment_recommendations[]`: from Step 4 if executed

**Compliance score**: `(passed / (total - info_only)) * 100`. Info-severity FAILs (A9, A10, A12, D7) don't reduce score.

#### 3.2 assessment-report.md
Markdown report: header → summary table → results by category → experiment recommendations.

#### 3.3 assessment-report.html
Single-file HTML with inline CSS. Color coding: green=PASS, red=FAIL, blue=INFO. Severity badges, collapsible sections, compliance score gauge.

#### 3.4 remediation-commands.sh
Executable script with fix commands for FAIL items only. Each section: check ID, explanation, command(s). **Never auto-executed.**

Present summary + compliance score. Ask if user wants Step 4.

### Cost Impact Assessment

Each FAIL includes a cost estimate.
- **With `awslabs.aws-pricing-mcp-server`**: Query actual on-demand pricing for compute/monitoring fixes, show monthly dollar estimates.
- **Without Pricing MCP (default)**: Qualitative descriptions (e.g., "Zero — config only" or "+1 Pod per workload").

---

### Step 4: Experiment Recommendations (Optional)

Map FAIL items to chaos experiments. See [fail-to-experiment-mapping.md](references/fail-to-experiment-mapping.md) for the complete mapping table.

**Key mappings**:

| Check FAIL | Fault Type | Priority | Hypothesis |
|------------|-----------|----------|------------|
| A1: Singleton Pod | pod_kill | P0 | Permanent loss until manual restart |
| A2: Single Replica | pod_kill | P0 | Service downtime ~30-60s |
| A3: No Anti-Affinity | node_terminate | P1 | Node loss may kill all replicas |
| D1: No Node Autoscaler | cpu_stress | P1 | Resource exhaustion blocks scheduling |
| D2: Single AZ | az_network_disrupt | P0 | Complete cluster unavailability |

For each FAIL with a mapping, generate experiment recommendation entries in `assessment.json`. Sort by priority (P0 > P1 > P2).

**Handoff**: Guide user to invoke `chaos-engineering-on-aws` with `output/assessment.json` as Method 3 input.

---

## Safety Principles

1. **Read-only only**: All checks use `get`, `list`, `describe` — no create/update/delete during assessment
2. **Remediation requires manual execution**: `remediation-commands.sh` is never auto-executed
3. **System namespace exclusion**: `kube-system`, `kube-public`, `kube-node-lease` excluded from workload checks
4. **No secret exposure**: Never read Secret/ConfigMap values — only check existence
5. **Fargate awareness**: Skip inapplicable checks for Fargate workloads
6. **EKS Auto Mode awareness**: Adjust node-related checks when Auto Mode detected

## Error Handling

| Error | Action |
|-------|--------|
| `kubectl` connection refused | Verify kubeconfig, check endpoint accessibility |
| AWS credential error | Run `aws sts get-caller-identity` |
| Permission denied | Mark check as `"SKIPPED"` with reason |
| Addon describe fails | Treat as "not managed" |
| Timeout on large cluster | Suggest narrowing target namespaces |

Never block entire assessment for a single check failure.

## Language

If the user speaks Chinese, respond in Chinese while still following the procedures above. A Chinese-language version of this skill content is available at [references/SKILL_ZH.md](references/SKILL_ZH.md) for reference.

## References

- [EKS-Resiliency-Checkpoints.md](references/EKS-Resiliency-Checkpoints.md) — What each check does and why
- [check-commands.md](references/check-commands.md) — How to execute each check (commands + PASS/FAIL)
- [eks-resiliency-checks-mcp.md](references/eks-resiliency-checks-mcp.md) — MCP-based check execution
- [remediation-templates.md](references/remediation-templates.md) — Fix command templates
- [fail-to-experiment-mapping.md](references/fail-to-experiment-mapping.md) — FAIL → chaos experiment mapping
- [eks-auth-setup.md](references/eks-auth-setup.md) — EKS authentication setup guide
- [examples/petsite-assessment.md](examples/petsite-assessment.md) — Sample assessment output
- [references/SKILL_ZH.md](references/SKILL_ZH.md) — Chinese-language version of this skill
