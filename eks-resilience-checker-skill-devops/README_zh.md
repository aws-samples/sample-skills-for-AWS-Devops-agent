中文 | [English](README.md)

# EKS 韧性检查器

AI 驱动的 Agent Skill，对 Amazon EKS 集群执行自动化韧性评估，覆盖 **应用工作负载**（A1-A14）、**控制平面**（C1-C5）、**数据平面**（D1-D7）共 26 项最佳实践检查。输出结构化结果，可直接作为 `chaos-engineering-on-aws` Skill 的输入驱动混沌实验。

## 在韧性生命周期中的定位

```
┌──────────────────────────────────────────────────────────────────────────────────────┐
│                         AWS 韧性生命周期框架                                          │
│                                                                                      │
│  阶段 1: 目标设定       阶段 2: 设计与实施          阶段 3: 评估与测试                  │
│  ┌───────────────────┐  ┌────────────────────────┐  ┌──────────────────────────────┐  │
│  │ aws-rma-           │  │ aws-resilience-         │  │ chaos-engineering-on-aws      │  │
│  │ assessment          │─>│ modeling                │─>│ 混沌实验 + 指标 + 日志分析    │  │
│  │ "我们在哪?"         │  │ "什么可能出错?"          │  │ "真的会坏吗?"                 │  │
│  └───────────────────┘  └────────────────────────┘  └───────────┬──────────────────┘  │
│                                    ^                              │                     │
│                                    │    ┌─────────────────────────┴──────────────────┐  │
│                                    │    │ eks-resilience-checker（本 Skill）           │  │
│                                    │    │ ① 26 项 K8s 韧性评估 → assessment.json      │  │
│                                    │    │ ② FAIL → 实验推荐 → chaos skill 消费        │  │
│                                    │    └────────────────────────────────────────────┘  │
│                                    └──────────── 反馈循环 ───────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────────────────────┘
```

| # | Skill | 生命周期阶段 | 输入 | 输出 |
|---|-------|-------------|------|------|
| 1 | **aws-rma-assessment** | 阶段 1: 目标设定 | 引导式问答 | 韧性成熟度评分 + 改进路线图 |
| 2 | **aws-resilience-modeling** | 阶段 2: 设计与实施 | AWS 账号 / 架构文档 | 风险清单 + 资源扫描 + 缓解策略 |
| 3 | **chaos-engineering-on-aws** | 阶段 3: 评估与测试 | Skill 2 报告 + Skill 4 评估 | 实验报告 + 日志分析 + 韧性验证 |
| 4 | **eks-resilience-checker** | 阶段 3: 评估与测试 | EKS 集群直连 | 26 项合规报告 + 实验推荐 |

## 安装

### 一键安装（推荐）

```bash
# 安装本 Skill 到你的 AI Agent
npx skills add aws-samples/sample-aws-resilience-skill --skill eks-resilience-checker

# 安装全部 4 个韧性 Skill
npx skills add aws-samples/sample-aws-resilience-skill --skill '*'

# 安装到指定 Agent
npx skills add aws-samples/sample-aws-resilience-skill --skill eks-resilience-checker -a claude-code

# 全局安装（跨项目可用）
npx skills add aws-samples/sample-aws-resilience-skill --skill eks-resilience-checker -g
```

### 手动安装

```bash
git clone https://github.com/aws-samples/sample-aws-resilience-skill.git
ln -s $(pwd)/sample-aws-resilience-skill/eks-resilience-checker ~/.claude/skills/eks-resilience-checker
```

## 快速开始

1. **配置 kubectl 访问** 你的 EKS 集群
2. **告诉你的 AI Agent**："对我的集群运行 EKS 韧性评估"
3. **查看** `output/` 目录中生成的报告

## 前置条件

| 工具 | 用途 | 必需 |
|------|------|------|
| `kubectl` | K8s API 查询 | 是 |
| `aws` CLI | EKS describe-cluster + addon 查询 | 是 |
| `jq` | JSON 解析 | 是 |
| EKS 集群访问 | kubectl 已配置目标集群 | 是 |

### MCP Server（可选增强）

| Server | 用途 |
|--------|------|
| `awslabs.eks-mcp-server` | K8s 资源查询（替代 kubectl） |

当 MCP 不可用时，回退到 `kubectl` + `aws` CLI 直接调用。

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

## 检查分类

### 应用检查（A1-A14）

| 编号 | 检查项 | 严重级别 |
|------|--------|---------|
| A1 | 避免 Singleton Pod | Critical |
| A2 | 多副本部署 | Critical |
| A3 | Pod Anti-Affinity | Warning |
| A4 | Liveness Probe | Critical |
| A5 | Readiness Probe | Critical |
| A6 | Pod Disruption Budget | Warning |
| A7 | Metrics Server | Warning |
| A8 | Horizontal Pod Autoscaler | Warning |
| A9 | Custom Metrics Scaling | Info |
| A10 | Vertical Pod Autoscaler | Info |
| A11 | PreStop Hook | Warning |
| A12 | Service Mesh | Info |
| A13 | 应用监控 | Warning |
| A14 | 集中日志 | Warning |

### 控制平面检查（C1-C5）

| 编号 | 检查项 | 严重级别 |
|------|--------|---------|
| C1 | 控制平面日志 | Warning |
| C2 | 集群认证 | Warning |
| C3 | 大规模集群优化 | Info |
| C4 | API Server 访问控制 | Critical |
| C5 | 避免 Catch-All Webhook | Warning |

### 数据平面检查（D1-D7）

| 编号 | 检查项 | 严重级别 |
|------|--------|---------|
| D1 | 节点自动伸缩 | Critical |
| D2 | 多 AZ 节点分布 | Critical |
| D3 | Resource Requests/Limits | Critical |
| D4 | Namespace ResourceQuota | Warning |
| D5 | Namespace LimitRange | Warning |
| D6 | CoreDNS Metrics 监控 | Warning |
| D7 | CoreDNS 托管配置 | Info |

## 输出文件

```
output/
├── assessment.json              # 结构化评估结果（26 项）— chaos skill 可消费
├── assessment-report.md         # 人类可读报告（Markdown）
├── assessment-report.html       # HTML 报告（内联 CSS，可独立打开）
└── remediation-commands.sh      # 一键修复脚本（可执行的 kubectl/aws 命令）
```

### assessment.json 结构

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

## 与 chaos-engineering-on-aws 的集成

`assessment.json` 作为 chaos skill 步骤 1 的 **方式 3** 输入：

```
chaos-engineering-on-aws 步骤 1 — 输入源：
  方式 1: aws-resilience-modeling 报告     → AWS 资源级风险
  方式 2: 独立 chaos-input 文件           → 手动指定
  方式 3: eks-resilience-checker 评估      → K8s 配置风险（新增）
```

chaos skill 读取 `assessment.json` 中的 `experiment_recommendations`，按优先级排序（P0 > P1 > P2），将每项映射到 `fault-catalog.yaml` 中的故障类型。

## 四步工作流

| 步骤 | 名称 | 输出 |
|------|------|------|
| 1 | 集群发现 | 集群元数据、namespace 列表 |
| 2 | 自动化检查（26 项） | 每项检查结果 |
| 3 | 生成报告 | `output/assessment.json` + `.md` + `.html` + `remediation-commands.sh` |
| 4 | 实验推荐（可选） | FAIL → 实验映射 |

## 示例报告

参见 [examples/petsite-assessment.md](examples/petsite-assessment.md) 了解 PetSite EKS 集群的评估报告样例。

## 目录结构

```
eks-resilience-checker/
├── SKILL.md                            # 主指令文件（含 DevOps Agent frontmatter）
├── README.md                           # 英文版
├── README_zh.md                        # 本文件（中文版）
├── doc/
│   └── prd.md                          # 产品需求文档
├── references/
│   ├── SKILL_ZH.md                     # SKILL.md 中文版（参考用）
│   ├── EKS-Resiliency-Checkpoints.md   # 26 项检查详细说明
│   ├── check-commands.md               # 每项检查对应的 kubectl/aws 命令
│   ├── eks-resiliency-checks-mcp.md    # 基于 MCP 的检查执行
│   ├── eks-auth-setup.md               # EKS 认证配置指南
│   ├── remediation-templates.md        # 修复命令模板
│   └── fail-to-experiment-mapping.md   # FAIL → 实验推荐映射表
└── examples/
    └── petsite-assessment.md           # PetSite 集群评估示例
```

## 与 AWS DevOps Agent 配合使用

本 Skill 已兼容 [AWS DevOps Agent Custom Skills](https://docs.aws.amazon.com/devopsagent/latest/userguide/about-aws-devops-agent-devops-agent-skills.html) 格式。

上传步骤：

1. 将整个 `eks-resilience-checker-skill-devops/` 目录压缩为 `.zip`（不超过 6 MB）。
2. 进入 DevOps Agent Operator Web App 的 Skills 页面。
3. 点击 **Add skill** → **Upload skill**。
4. 拖入 zip 文件，选择目标 agent type（默认 Generic 适用所有类型），点击 **Upload**。

DevOps Agent 会在用户请求匹配 `SKILL.md` frontmatter 中声明的触发词（如 "EKS resilience"、"集群评估"、"韧性评估"）时自动加载本 Skill。
