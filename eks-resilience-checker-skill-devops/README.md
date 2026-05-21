[**中文**](README_zh.md) | English

# EKS Resilience Checker

An AI-powered Agent Skill that performs automated resilience assessment of Amazon EKS clusters against 26 best practice checks covering **Application workloads** (A1-A14), **Control Plane** (C1-C5), and **Data Plane** (D1-D7). Outputs structured results that can directly feed into the `chaos-engineering-on-aws` skill to drive chaos experiments.

## How It Fits — Resilience Lifecycle

```
┌──────────────────────────────────────────────────────────────────────────────────────┐
│                         AWS Resilience Lifecycle Framework                            │
│                                                                                      │
│  Stage 1: Set Objectives  Stage 2: Design & Implement  Stage 3: Evaluate & Test      │
│  ┌───────────────────┐    ┌────────────────────────┐   ┌──────────────────────────┐  │
│  │ aws-rma-           │    │ aws-resilience-         │   │ chaos-engineering-on-aws  │  │
│  │ assessment          │───>│ modeling                │──>│ Chaos experiments +       │  │
│  │ "Where are we?"    │    │ "What could go wrong?"  │   │ metrics + log analysis    │  │
│  └───────────────────┘    └────────────────────────┘   └───────────┬──────────────┘  │
│                                      ^                              │                 │
│                                      │    ┌─────────────────────────┴──────────────┐  │
│                                      │    │ eks-resilience-checker (this Skill)     │  │
│                                      │    │ 1. 26 K8s resilience checks            │  │
│                                      │    │ 2. FAIL -> experiment recommendations  │  │
│                                      │    └────────────────────────────────────────┘  │
│                                      └──────────── Feedback Loop ────────────────────┘ │
└──────────────────────────────────────────────────────────────────────────────────────┘
```

| # | Skill | Lifecycle Stage | Input | Output |
|---|-------|----------------|-------|--------|
| 1 | **aws-rma-assessment** | Stage 1: Set Objectives | Guided Q&A | Resilience maturity score + roadmap |
| 2 | **aws-resilience-modeling** | Stage 2: Design & Implement | AWS account / architecture docs | Risk inventory + resource scan + mitigations |
| 3 | **chaos-engineering-on-aws** | Stage 3: Evaluate & Test | Skill 2 report + Skill 4 assessment | Experiment report + log analysis + validation |
| 4 | **eks-resilience-checker** | Stage 3: Evaluate & Test | Direct EKS cluster access | 26-check compliance report + experiment recommendations |

## Installation

### One-Line Install (Recommended)

```bash
# Install this skill to your AI agent
npx skills add aws-samples/sample-aws-resilience-skill --skill eks-resilience-checker

# Install all 4 resilience skills
npx skills add aws-samples/sample-aws-resilience-skill --skill '*'

# Install to a specific agent
npx skills add aws-samples/sample-aws-resilience-skill --skill eks-resilience-checker -a claude-code

# Global install (available across projects)
npx skills add aws-samples/sample-aws-resilience-skill --skill eks-resilience-checker -g
```

### Manual Install

```bash
git clone https://github.com/aws-samples/sample-aws-resilience-skill.git
ln -s $(pwd)/sample-aws-resilience-skill/eks-resilience-checker ~/.claude/skills/eks-resilience-checker
```

## Quick Start

1. **Configure kubectl access** to your EKS cluster
2. **Tell your AI agent**: "Run EKS resilience assessment on my cluster"
3. **Review** the generated report in `output/`

## Prerequisites

| Tool | Purpose | Required |
|------|---------|----------|
| `kubectl` | K8s API queries | Yes |
| `aws` CLI | EKS describe-cluster + addon queries | Yes |
| `jq` | JSON parsing | Yes |
| EKS cluster access | kubectl configured with target cluster | Yes |

### MCP Server (Optional Enhancement)

| Server | Purpose |
|--------|---------|
| `awslabs.eks-mcp-server` | K8s resource queries (alternative to kubectl) |

When MCP is unavailable, the skill falls back to `kubectl` + `aws` CLI.

```json
{
  "mcpServers": {
    "awslabs.eks-mcp-server": {
      "command": "uvx",
      "args": ["awslabs.eks-mcp-server@latest"],
      "env": { "AWS_REGION": "ap-northeast-1", "FASTMCP_LOG_LEVEL": "ERROR" }
    }
  }
}
```

## Check Categories

### Application Checks (A1-A14)

| ID | Check | Severity |
|----|-------|----------|
| A1 | Avoid Singleton Pods | Critical |
| A2 | Run Multiple Replicas | Critical |
| A3 | Use Pod Anti-Affinity | Warning |
| A4 | Use Liveness Probes | Critical |
| A5 | Use Readiness Probes | Critical |
| A6 | Use Pod Disruption Budgets | Warning |
| A7 | Run Kubernetes Metrics Server | Warning |
| A8 | Use Horizontal Pod Autoscaler | Warning |
| A9 | Use Custom Metrics Scaling | Info |
| A10 | Use Vertical Pod Autoscaler | Info |
| A11 | Use PreStop Hooks | Warning |
| A12 | Use a Service Mesh | Info |
| A13 | Monitor Your Applications | Warning |
| A14 | Use Centralized Logging | Warning |

### Control Plane Checks (C1-C5)

| ID | Check | Severity |
|----|-------|----------|
| C1 | Monitor Control Plane Logs | Warning |
| C2 | Cluster Authentication | Warning |
| C3 | Running Large Clusters | Info |
| C4 | API Server Endpoint Access Control | Critical |
| C5 | Avoid Catch-All Admission Webhooks | Warning |

### Data Plane Checks (D1-D7)

| ID | Check | Severity |
|----|-------|----------|
| D1 | Use Cluster Autoscaler or Karpenter | Critical |
| D2 | Worker Nodes Spread Across Multiple AZs | Critical |
| D3 | Configure Resource Requests/Limits | Critical |
| D4 | Namespace ResourceQuotas | Warning |
| D5 | Namespace LimitRanges | Warning |
| D6 | Monitor CoreDNS Metrics | Warning |
| D7 | CoreDNS Managed Configuration | Info |

## Output Files

```
output/
├── assessment.json              # Structured results (26 checks) — consumable by chaos skill
├── assessment-report.md         # Human-readable report (Markdown)
├── assessment-report.html       # HTML report (inline CSS, standalone)
└── remediation-commands.sh      # One-click fix script (executable kubectl/aws commands)
```

### assessment.json Structure

```json
{
  "schema_version": "1.0",
  "cluster_name": "my-cluster",
  "region": "ap-northeast-1",
  "kubernetes_version": "1.32",
  "timestamp": "2026-04-03T08:00:00Z",
  "summary": {
    "total_checks": 28,
    "passed": 20,
    "failed": 6,
    "info": 2,
    "compliance_score": 71.4
  },
  "checks": [
    {
      "id": "A2",
      "name": "Run Multiple Replicas",
      "category": "application",
      "severity": "critical",
      "status": "FAIL",
      "findings": ["..."],
      "resources_affected": ["..."],
      "remediation": "Set spec.replicas > 1 for all production workloads.",
      "cost_impact": "+1 Pod per workload — doubles CPU/memory; may trigger additional node"
    }
  ],
  "experiment_recommendations": [ ... ]
}
```

## Integration with chaos-engineering-on-aws

The `assessment.json` output serves as **Method 3** input for the chaos skill's Step 1:

```
chaos-engineering-on-aws Step 1 — Input Sources:
  Method 1: aws-resilience-modeling report     → AWS resource-level risks
  Method 2: Standalone chaos-input file        → Manual specification
  Method 3: eks-resilience-checker assessment  → K8s configuration risks (NEW)
```

The chaos skill reads `experiment_recommendations` from `assessment.json`, sorts by priority (P0 > P1 > P2), and maps each to fault types in `fault-catalog.yaml`.

## Four-Step Workflow

| Step | Name | Output |
|------|------|--------|
| 1 | Cluster Discovery | Cluster metadata, namespace list |
| 2 | Automated Checks (26 items) | Per-check findings |
| 3 | Generate Reports | `output/assessment.json` + `.md` + `.html` + `remediation-commands.sh` |
| 4 | Experiment Recommendations (optional) | FAIL-to-experiment mapping |

## Example Report

See [examples/petsite-assessment.md](examples/petsite-assessment.md) for a sample assessment report on a PetSite EKS cluster.

## Directory Structure

```
eks-resilience-checker/
├── SKILL.md                            # Main skill instructions (with DevOps Agent frontmatter)
├── README.md                           # This file (English)
├── README_zh.md                        # Chinese version
├── doc/
│   └── prd.md                          # Product requirements
├── references/
│   ├── SKILL_ZH.md                     # Chinese-language version of SKILL.md
│   ├── EKS-Resiliency-Checkpoints.md   # 26-check detailed descriptions
│   ├── check-commands.md               # kubectl/aws commands per check
│   ├── eks-resiliency-checks-mcp.md    # MCP-based check execution
│   ├── eks-auth-setup.md               # EKS authentication setup guide
│   ├── remediation-templates.md        # Fix command templates
│   └── fail-to-experiment-mapping.md   # FAIL → experiment mapping table
└── examples/
    └── petsite-assessment.md           # PetSite cluster assessment example
```

## Use with AWS DevOps Agent

This skill is compatible with the [AWS DevOps Agent Custom Skills](https://docs.aws.amazon.com/devopsagent/latest/userguide/about-aws-devops-agent-devops-agent-skills.html) format.

To use this skill with AWS DevOps Agent:

1. Compress the entire `eks-resilience-checker-skill-devops/` directory into a `.zip` file (max 6 MB).
2. In your DevOps Agent Operator Web App, navigate to the Skills page.
3. Click **Add skill** → **Upload skill**.
4. Drag the zip file in, choose the agent types that should use this skill (Generic by default), and click **Upload**.

The DevOps Agent will automatically load this skill when a user request matches the triggers declared in the `SKILL.md` frontmatter (e.g. "EKS resilience", "cluster assessment", "韧性评估").
