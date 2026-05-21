<!-- AGENT: Do not load this file during assessment execution. This is a design document for human reference only. -->

# 📄 Product Requirement Document (PRD)
## EKS Resilience Checker — Agent Skill

**版本**: 0.2  
**日期**: 2026-04-03  
**状态**: Draft  
**仓库**: `sample-aws-resilience-skill/eks-resilience-checker`  
**变更**: v0.2 — 日志分析移至 chaos-engineering-on-aws；聚焦 26 项评估；assessment.json → chaos skill 集成接口  
**变更**: v0.3 — 新增分发安装方式（`npx skills add`）；更新 Lifecycle 图加入第四个 Skill  
**变更**: v0.4 — Section 10 改为"从未合并 PR 借鉴的内容"，去掉 MCP Server 方案，聚焦 Skill 本身

---

## 1. Executive Summary

### 1.1 产品愿景

开发一个面向 **Claude Code / Kiro CLI / Codex** 的 Agent Skill，对 Amazon EKS 集群执行全面的韧性架构评估。覆盖三层：**应用工作负载**（A1-A14）、**控制平面**（C1-C5）、**数据平面**（D1-D7），共 26 项检查。输出结构化评估结果，可直接作为 `chaos-engineering-on-aws` 的输入驱动混沌实验。

### 1.2 在 Resilience Lifecycle 中的定位

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                         AWS Resilience Lifecycle Framework                                │
│                                                                                          │
│  Stage 1: Set Objectives  Stage 2: Design & Implement  Stage 3: Evaluate & Test          │
│  ┌───────────────────┐    ┌────────────────────────┐   ┌──────────────────────────────┐  │
│  │ aws-rma-           │    │ aws-resilience-         │   │ chaos-engineering-on-aws      │  │
│  │ assessment          │───►│ modeling                │──►│ 混沌实验 + 指标 + 日志分析    │  │
│  │ "我们在哪?"         │    │ "什么可能出错?"          │   │ "真的会坏吗?"                 │  │
│  └───────────────────┘    └────────────────────────┘   └───────────┬──────────────────┘  │
│                                      ▲                              │                     │
│                                      │                              │                     │
│                                      │    ┌─────────────────────────┴──────────────────┐  │
│                                      │    │ eks-resilience-checker (本 Skill, 第 4 个)   │  │
│                                      │    │ ① 26 项 K8s 韧性评估 → assessment.json      │  │
│                                      │    │ ② FAIL → 实验推荐 → chaos skill 消费        │  │
│                                      │    └────────────────────────────────────────────┘  │
│                                      └──────────── Feedback Loop ────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

**四个 Skill 完整链路**：

| # | Skill | Lifecycle Stage | 输入 | 输出 |
|---|-------|----------------|------|------|
| 1 | **aws-rma-assessment** | Stage 1: Set Objectives | 引导式问答 | 韧性成熟度评分 + 改进路线图 |
| 2 | **aws-resilience-modeling** | Stage 2: Design & Implement | AWS 账号 / 架构文档 | 风险清单 + 资源扫描 + 缓解策略 |
| 3 | **chaos-engineering-on-aws** | Stage 3: Evaluate & Test | Skill 2 报告 + Skill 4 评估 | 实验报告 + 日志分析 + 韧性验证 |
| 4 | **eks-resilience-checker** | Stage 3: Evaluate & Test | EKS 集群直连 | 26 项合规报告 + 实验推荐 |

**一句话定义**：自动化评估 EKS 集群 26 项韧性最佳实践，输出结构化结果供混沌实验消费。

### 1.3 职责边界

| 功能 | eks-resilience-checker | chaos-engineering-on-aws |
|------|----------------------|--------------------------|
| 26 项 K8s 配置评估 | ✅ 本 Skill | ❌ |
| 评估报告 + 修复脚本 | ✅ 本 Skill | ❌ |
| FAIL → 实验推荐映射 | ✅ 本 Skill 输出 | ✅ Step 1 消费 |
| 混沌实验执行 | ❌ | ✅ |
| CloudWatch 指标监控 | ❌ | ✅ Step 5 已有 |
| 实时应用日志分析 | ❌ | ✅ Step 5 扩展（新增） |
| 事后日志分析 | ❌ | ✅ Step 6 扩展（新增） |

### 1.4 核心价值

| 没有这个 Skill | 有了这个 Skill |
|---------------|---------------|
| 手动逐项检查 EKS 配置，耗时且容易遗漏 | 自动化 26 项检查，5 分钟出报告 |
| 混沌实验选目标靠经验 | FAIL 项自动映射到推荐实验场景 |
| 实验前不知道哪些服务缺少 PDB / Probe / Anti-Affinity | 自动识别韧性短板，量化合规分数 |
| AWS 资源风险和 K8s 配置风险脱节 | 两个 Skill 的评估结果合流到 chaos-engineering-on-aws |

### 1.5 目标用户

| 角色 | 使用场景 | 技术水平 |
|------|---------|---------|
| **AWS 架构师** | 为客户做 EKS 韧性评估、输出改进报告 | 精通 AWS + K8s |
| **SRE / DevOps** | 生产集群韧性巡检、混沌实验前基线 | 精通 K8s 运维 |
| **开发团队 Lead** | 了解应用韧性短板、推动改进 | 了解 K8s 基础 |

### 1.6 使用工具

| 工具 | 说明 |
|------|------|
| **Claude Code** | 主要使用环境 |
| **Kiro CLI** | 备选使用环境 |
| **Codex CLI** | 备选使用环境 |

---

## 2. 功能范围

### 2.1 EKS 韧性基线评估（26 项检查）

自动化评估 EKS 集群的韧性配置，覆盖三个层面：

#### Application Checks（A1-A14）

| 编号 | 检查项 | 检查手段 | 严重级别 |
|------|--------|---------|---------|
| A1 | 避免 Singleton Pod | `kubectl get pods` + 检查 ownerReferences | 🔴 Critical |
| A2 | 多副本部署 | Deployment/StatefulSet replicas > 1 | 🔴 Critical |
| A3 | Pod Anti-Affinity | 多副本 Deployment 是否配置 podAntiAffinity | 🟡 Warning |
| A4 | Liveness Probe | 所有容器是否配置 livenessProbe | 🔴 Critical |
| A5 | Readiness Probe | 所有容器是否配置 readinessProbe | 🔴 Critical |
| A6 | Pod Disruption Budget | 关键工作负载是否有 PDB | 🟡 Warning |
| A7 | Metrics Server | kube-system 中 metrics-server 是否运行 | 🟡 Warning |
| A8 | Horizontal Pod Autoscaler | 多副本工作负载是否有 HPA | 🟡 Warning |
| A9 | Custom Metrics Scaling | 是否有 KEDA / Prometheus Adapter 等自定义指标 | 🟢 Info |
| A10 | Vertical Pod Autoscaler | VPA CRD + Controller 是否安装使用 | 🟢 Info |
| A11 | PreStop Hook | Deployment/StatefulSet 是否配置 preStop lifecycle | 🟡 Warning |
| A12 | Service Mesh | 是否有 Istio / Linkerd / Consul | 🟢 Info |
| A13 | 应用监控 | Prometheus / CloudWatch Container Insights / Datadog 等 | 🟡 Warning |
| A14 | 集中日志 | Fluent Bit / CloudWatch Logs / Loki 等 | 🟡 Warning |

#### Control Plane Checks（C1-C5）

| 编号 | 检查项 | 检查手段 | 严重级别 |
|------|--------|---------|---------|
| C1 | 控制平面日志 | `aws eks describe-cluster` logging config | 🟡 Warning |
| C2 | 集群认证 | EKS Access Entries 或 aws-auth ConfigMap | 🟡 Warning |
| C3 | 大规模集群优化 | >1000 services 时 IPVS + VPC CNI 缓存 | 🟢 Info |
| C4 | API Server 访问控制 | endpoint public/private + CIDR 限制 | 🔴 Critical |
| C5 | 避免 Catch-All Webhook | MutatingWebhook / ValidatingWebhook 范围检查 | 🟡 Warning |

#### Data Plane Checks（D1-D7）

| 编号 | 检查项 | 检查手段 | 严重级别 |
|------|--------|---------|---------|
| D1 | 节点自动伸缩 | Cluster Autoscaler 或 Karpenter | 🔴 Critical |
| D2 | 多 AZ 节点分布 | 节点 AZ label 分布 + 均衡性（±20%） | 🔴 Critical |
| D3 | Resource Requests/Limits | 所有 Deployment 容器资源规格 | 🔴 Critical |
| D4 | Namespace ResourceQuota | 用户 namespace 是否有 ResourceQuota | 🟡 Warning |
| D5 | Namespace LimitRange | 用户 namespace 是否有 LimitRange | 🟡 Warning |
| D6 | CoreDNS Metrics 监控 | CoreDNS metrics port + ServiceMonitor | 🟡 Warning |
| D7 | CoreDNS 托管配置 | EKS Managed Add-on 还是自管理 | 🟢 Info |

#### 评估输出

```
output/
├── assessment.json              # 结构化评估结果（26 项）— chaos skill 可消费
├── assessment-report.md         # 人类可读报告（Markdown）
├── assessment-report.html       # HTML 报告（内联 CSS，可独立打开）
└── remediation-commands.sh      # 一键修复脚本（可执行的 kubectl/aws 命令）
```

### 2.2 assessment.json 结构化输出

```json
{
  "schema_version": "1.0",
  "cluster_name": "PetSite",
  "region": "ap-northeast-1",
  "kubernetes_version": "1.32",
  "platform_version": "eks.8",
  "timestamp": "2026-04-03T08:00:00Z",
  "target_namespaces": ["petadoptions", "default"],

  "summary": {
    "total_checks": 28,
    "passed": 20,
    "failed": 6,
    "info": 2,
    "critical_failures": 3,
    "compliance_score": 71.4
  },

  "checks": [
    {
      "id": "A2",
      "name": "Run Multiple Replicas",
      "category": "application",
      "severity": "critical",
      "status": "FAIL",
      "findings": [
        "Deployment petadoptions/payforadoption has replicas=1",
        "Deployment petadoptions/pethistory has replicas=1"
      ],
      "resources_affected": [
        "petadoptions/payforadoption",
        "petadoptions/pethistory"
      ],
      "remediation": "kubectl scale deployment payforadoption --replicas=2 -n petadoptions",
      "chaos_experiment_recommendation": {
        "description": "Kill single-replica pod to measure actual downtime",
        "fault_types": ["pod_kill", "fis_eks_pod_delete"],
        "priority": "P0",
        "rationale": "Single replica = guaranteed downtime on pod failure"
      }
    }
  ],

  "experiment_recommendations": [
    {
      "priority": "P0",
      "check_id": "A2",
      "target_resources": ["petadoptions/payforadoption"],
      "suggested_fault_type": "pod_kill",
      "suggested_backend": "chaosmesh",
      "hypothesis": "Killing the single-replica payforadoption pod will cause service unavailability until K8s recreates the pod (~30-60s)",
      "expected_rto_seconds": 60
    }
  ]
}
```

### 2.3 FAIL → 实验推荐映射表

| 检查 FAIL | 推荐实验 | fault_catalog.yaml 类型 | 优先级 | 验证目标 |
|-----------|---------|----------------------|--------|---------|
| A1: Singleton Pod | Pod kill | `pod_kill` | P0 | 验证无控制器 Pod 是否真的无法恢复 |
| A2: 单副本 | Pod kill/delete | `pod_kill` / `fis_eks_pod_delete` | P0 | 测量单副本服务实际中断时长 |
| A3: 无 Anti-Affinity | 节点终止 | `fis_eks_terminate_node` | P1 | 验证所有副本是否在同一节点 |
| A4: 无 Liveness Probe | CPU stress | `pod_cpu_stress` | P1 | 验证无 probe 时僵尸进程是否被清理 |
| A5: 无 Readiness Probe | Network delay | `network_delay` | P1 | 验证无 readiness 时流量是否仍路由到异常 Pod |
| A6: 无 PDB | 节点终止 | `fis_eks_terminate_node` | P1 | 验证节点 drain 是否同时驱逐所有副本 |
| A8: 无 HPA | CPU stress | `pod_cpu_stress` | P2 | 验证高负载时是否无法自动扩容 |
| D1: 无节点伸缩 | CPU stress (全节点) | `fis_ssm_cpu_stress` | P1 | 验证节点资源耗尽后新 Pod 能否调度 |
| D2: 单 AZ | AZ 网络中断 | `fis_network_disrupt` / `fis_scenario_az_power_interruption` | P0 | 验证单 AZ 故障是否导致全集群不可用 |
| D3: 无 Resource Limits | Memory stress | `pod_memory_stress` | P1 | 验证是否影响同节点其他 Pod（noisy neighbor） |

---

## 3. 与 chaos-engineering-on-aws 的集成

### 3.1 作为第三输入源

chaos-engineering-on-aws 的 Step 1 "Define Experiment Targets" 增加输入方式：

```
Step 1 输入（三选一或组合）:
  Method 1: aws-resilience-modeling 报告     → AWS 资源级风险
  Method 2: 独立 chaos-input 文件           → 手动指定
  Method 3: eks-resilience-checker 的 assessment.json → K8s 配置风险 (新增)
```

**消费方式**：
1. 读取 `assessment.json` 的 `experiment_recommendations` 数组
2. 按 priority 排序（P0 > P1 > P2）
3. 每个推荐包含 `suggested_fault_type`（对应 `fault-catalog.yaml`）和 `target_resources`
4. 结合 Method 1 的 AWS 风险，合并去重后输出给用户确认

### 3.2 chaos-engineering-on-aws 日志分析扩展

日志分析功能作为 chaos-engineering-on-aws 的增强（不在本 Skill 中实现）：

```
chaos-engineering-on-aws Step 5 (改造后):
  Phase 2: Observation
    ├── monitor.sh → CloudWatch metrics       (已有)
    └── log-collector.sh → kubectl logs -f    (新增: 实时日志采集)
  Phase 3: Recovery
    └── 日志分析 + 错误分类                    (新增: timeout/connection/5xx/oom/other)

chaos-engineering-on-aws Step 6 (改造后):
  ├── 指标分析 (已有)
  ├── 应用日志分析章节 (新增)
  └── 事后日志分析独立入口 (新增: 用户给报告路径 → 直接做事后分析)
```

参考实现：
- 错误分类逻辑 → `graph-driven-chaos/code/runner/log_collector.py`
- 实时/事后双模式 + 交互设计 → `panlm/skills/eks-app-log-analysis`

---

## 4. 技术架构

### 4.1 工具依赖

| 工具 | 用途 | 必需 |
|------|------|------|
| `kubectl` | K8s API 查询 | ✅ |
| `aws` CLI | EKS describe-cluster + addon 查询 | ✅ |
| `jq` | JSON 解析 | ✅ |

#### MCP Server（可选增强）

| Server | 用途 |
|--------|------|
| `awslabs.eks-mcp-server` | K8s 资源查询（替代 kubectl） |

当 MCP 不可用时，回退到 `kubectl` + `aws` CLI 直接调用。

### 4.2 EKS 认证方式

支持三种 kubectl → EKS 集群的认证路径：

| 方式 | 命令 | 适用场景 | 运行时依赖 |
|------|------|---------|-----------|
| **已有 kubeconfig** | `export KUBECONFIG=/path/to/admin-kubeconfig` | 集群管理员已提供 kubeconfig | 无额外依赖 |
| **IAM kubeconfig** | `aws eks update-kubeconfig --name {CLUSTER} --region {REGION}` | 标准 EKS 开发环境 | AWS CLI + IAM 凭证 |
| **ServiceAccount Token** | 创建只读 SA → 生成自包含 kubeconfig | CI/CD 流水线、受限环境 | 无（首次设置需管理员） |

#### 已有 kubeconfig（最常用）

```bash
# 直接使用管理员提供的 kubeconfig
export KUBECONFIG=/path/to/admin-kubeconfig
kubectl get nodes
```

#### IAM kubeconfig

```bash
aws eks update-kubeconfig --name {CLUSTER_NAME} --region {REGION}
```
每次 kubectl 请求时自动通过 `aws eks get-token` 获取短期令牌。要求 IAM 身份在集群中有映射（EKS Access Entries 或 `aws-auth` ConfigMap）。

#### ServiceAccount Token（受限环境）

```bash
# 创建只读 ServiceAccount
kubectl create serviceaccount eks-resilience-checker -n kube-system
kubectl create clusterrolebinding eks-resilience-checker-readonly \
  --clusterrole=view \
  --serviceaccount=kube-system:eks-resilience-checker

# 生成令牌（有效期 1 年）
TOKEN=$(kubectl create token eks-resilience-checker -n kube-system --duration=8760h)

# 生成自包含 kubeconfig（无需 AWS CLI 即可使用）
ENDPOINT=$(aws eks describe-cluster --name {CLUSTER_NAME} --query 'cluster.endpoint' --output text)
CA_DATA=$(aws eks describe-cluster --name {CLUSTER_NAME} --query 'cluster.certificateAuthority.data' --output text)
kubectl config set-cluster eks-check --server=$ENDPOINT --certificate-authority-data=$CA_DATA --embed-certs=true --kubeconfig=./eks-resilience-kubeconfig
kubectl config set-credentials eks-checker --token=$TOKEN --kubeconfig=./eks-resilience-kubeconfig
kubectl config set-context eks-check --cluster=eks-check --user=eks-checker --kubeconfig=./eks-resilience-kubeconfig
kubectl config use-context eks-check --kubeconfig=./eks-resilience-kubeconfig
```

**运行时零 AWS 依赖**，只需 `kubectl` + 这个 kubeconfig 文件。适合 CI/CD 定期扫描。
使用 `view` ClusterRole（只读），符合本 Skill 纯只读评估的安全原则。

> **Skill 执行时**：Step 1 应先询问用户 "你有现成的 kubeconfig 吗？还是需要生成一个？"

参考来源：[RadiumGu/Chaosmesh-MCP](https://github.com/RadiumGu/Chaosmesh-MCP) EKS Authentication Methods

### 4.3 执行流程

```
Step 1: 集群发现
  ├── 用户提供 cluster name（或自动检测 current-context）
  ├── aws eks describe-cluster → 版本、VPC、endpoint、logging、addons
  └── 确认目标 namespace 列表（排除 kube-system 等系统 namespace）

Step 2: 自动化检查（26 项）
  ├── Application checks (A1-A14): kubectl 查询工作负载配置
  ├── Control Plane checks (C1-C5): aws eks API + kubectl 查询
  └── Data Plane checks (D1-D7): kubectl 查询节点 + 资源配置

Step 3: 生成报告
  ├── assessment.json — 结构化结果（chaos skill 可消费）
  ├── assessment-report.md — 人类可读报告
  ├── assessment-report.html — 带颜色的 HTML 报告
  └── remediation-commands.sh — 修复脚本

Step 4: 实验推荐（可选）
  ├── 基于 FAIL 项 + 映射表 → 生成 experiment_recommendations
  ├── 展示推荐列表，用户确认
  └── 输出供 chaos-engineering-on-aws Step 1 消费的格式
```

**用户交互**：
- Step 1: 确认集群和 namespace
- Step 3: 查看报告，选择是否生成修复脚本
- Step 4: 确认是否继续混沌实验（如果是 → 引导用户用 chaos-engineering-on-aws）

### 4.4 Skill 文件结构

```
eks-resilience-checker/
├── SKILL.md                            # 入口（语言检测 → 分流）
├── SKILL_EN.md                         # 英文版完整指令
├── SKILL_ZH.md                         # 中文版完整指令
├── README.md                           # 项目说明
├── README_zh.md                        # 中文说明
├── doc/
│   └── prd.md                          # 本文档
├── references/
│   ├── EKS-Resiliency-Checkpoints.md   # 26 项检查详细说明（已有）
│   ├── check-commands.md               # 每项检查对应的 kubectl/aws 命令
│   ├── remediation-templates.md        # 修复命令模板
│   └── fail-to-experiment-mapping.md   # FAIL → 实验推荐映射表
├── scripts/
│   └── assess.sh                       # 评估主脚本（可独立运行）
└── examples/
    └── petsite-assessment.md           # PetSite 集群评估示例
```

---

## 5. 安全和约束

### 5.1 权限要求

| 操作 | 最小权限 |
|------|---------|
| K8s 读取 | `get`, `list` on pods/deployments/statefulsets/daemonsets/services/nodes/pdb/hpa/vpa/webhooks/resourcequotas/limitranges/configmaps |
| EKS API | `eks:DescribeCluster`, `eks:ListAddons`, `eks:DescribeAddon`, `eks:ListAccessEntries` |

### 5.2 安全原则

1. **纯只读**：评估阶段不做任何写操作
2. **修复脚本需确认**：`remediation-commands.sh` 生成后需用户手动执行
3. **Namespace 隔离**：默认排除 `kube-system`、`kube-public`、`kube-node-lease`
4. **敏感信息**：不在报告中暴露 Secret / ConfigMap 的值，只检查存在性

### 5.3 已知限制

| 限制 | 影响 | 缓解 |
|------|------|------|
| 大规模集群（>100 namespace）评估耗时长 | 可能超过 Agent 超时 | 支持指定 namespace 列表 |
| EKS Auto Mode 部分检查逻辑不同 | D7 CoreDNS 需特殊处理 | 检测 auto mode → 调整检查逻辑 |
| Fargate 工作负载不适用部分检查 | A3 anti-affinity、D1 节点伸缩 等 | 检测 Fargate profile → 跳过不适用检查 |

### 5.4 设计决策

| # | 决策 | 理由 |
|---|------|------|
| D1 | A9（Custom Metrics）和 A12（Service Mesh）为 Info 而非 FAIL | 很多合理的集群不需要这些，标 FAIL 会拉低合规分数 |
| D2 | 日志分析放在 chaos-engineering-on-aws 而非本 Skill | 日志分析和实验执行天然在同一流程，不需要跨 Skill 传递上下文 |
| D3 | 支持 EKS Auto Mode | D7 检查自动 PASS（CoreDNS 由平台管理）；节点相关检查按 Auto Mode 调整 |
| D4 | 不支持 Fargate 专项检查 | Fargate 是计算形态选择，不是韧性问题。检测到 Fargate → 跳过不适用检查 |
| D5 | 评估结果只生成本地文件，不持久化到 DynamoDB/S3 | 保持 Skill 简单，用户需要持久化可自行上传 |
| D6 | LLM 不参与检查逻辑 | 26 项检查是确定性规则，不需要 LLM 判断。LLM 只负责生成报告叙述和改进建议 |

---

## 6. 输出语言规则

- 用户说英文 → 英文输出
- 用户说中文 → 中文输出
- SKILL.md 入口检测语言后分流到 SKILL_EN.md / SKILL_ZH.md

---

## 7. 里程碑

| 阶段 | 内容 | 交付物 |
|------|------|--------|
| **M1: 核心评估** | 26 项检查 + JSON/MD 报告 | SKILL.md + assess.sh + references/ |
| **M2: 修复脚本** | remediation-commands.sh + HTML 报告 | scripts/ + 报告增强 |
| **M3: 实验推荐** | FAIL → 实验映射 + assessment.json 集成接口 | fail-to-experiment-mapping.md |
| **M4: chaos 集成** | chaos-engineering-on-aws Step 1 消费 assessment.json | chaos skill 侧改动 |
| **M5: 分发上线** | `npx skills add` 支持 + README + 示例 | SKILL.md frontmatter + examples/ |

---

## 8. chaos-engineering-on-aws 日志分析扩展（单独跟踪）

以下内容不在本 Skill 范围内，但作为 chaos-engineering-on-aws 的增强需求记录：

### 8.1 需求

在 chaos-engineering-on-aws 的 Step 5/6 中增加应用日志分析能力：

| 功能 | 来源参考 | 实现位置 |
|------|---------|---------|
| 实时 Pod 日志采集 | `log_collector.py` + `eks-app-log-analysis` | Step 5 Phase 2 新增 log-collector.sh |
| 错误分类（5 类） | `log_collector.py` | Step 5 Phase 3 |
| 按服务分组分析 | `eks-app-log-analysis` | Step 6 报告新增章节 |
| 跨服务关联时间线 | `eks-app-log-analysis` | Step 6 报告新增章节 |
| 事后日志分析入口 | `eks-app-log-analysis` | Step 6 独立入口 |

### 8.2 实现要点

- `monitor.sh` 和 `log-collector.sh` 并行运行（metrics + logs 双通道）
- 错误分类：timeout / connection / 5xx / oom / other
- 日志采集使用 `kubectl logs -f -l app={label} --prefix --timestamps`
- 事后模式：`kubectl logs --since-time={start} deployment/{name}`
- 报告中加入"应用日志分析"章节：错误时间线 + 错误模式 + 恢复检测 + 跨服务关联

---

## 9. 分发和安装

### 9.1 `npx skills add` 一键安装

本 Skill 通过 [vercel-labs/skills](https://github.com/vercel-labs/skills) CLI 分发，用户可以一键安装到任何支持的 Agent（Claude Code / Codex / Kiro / OpenCode / Cursor 等 40+ 种）。

#### 安装命令

```bash
# 列出仓库中所有可用 Skill
npx skills add panlm/sample-aws-resilience-skill --list

# 安装单个 Skill
npx skills add panlm/sample-aws-resilience-skill --skill eks-resilience-checker

# 安装所有 Skill（全套 4 个）
npx skills add panlm/sample-aws-resilience-skill --skill '*'

# 安装到指定 Agent
npx skills add panlm/sample-aws-resilience-skill --skill eks-resilience-checker -a claude-code -a codex

# 全局安装（跨项目可用）
npx skills add panlm/sample-aws-resilience-skill --skill eks-resilience-checker -g

# CI/CD 友好的非交互安装
npx skills add panlm/sample-aws-resilience-skill --skill eks-resilience-checker -a claude-code -g -y
```

#### 安装原理

`npx skills add` 的工作机制：

1. **从 GitHub 仓库拉取** Skill 目录（不是 npm 包，是 Git 仓库）
2. **识别 SKILL.md** 的 frontmatter（`name` + `description`）确定 Skill 名称
3. **检测本地已安装的 Agent**（Claude Code → `.claude/skills/`、Codex → `.agents/skills/` 等）
4. **Symlink**（默认）或 Copy 到对应 Agent 的 skills 目录
5. Agent 下次运行时自动加载新 Skill

```
仓库结构:
sample-aws-resilience-skill/
├── aws-rma-assessment/SKILL.md          → npx skills 识别为 "aws-rma-assessment"
├── aws-resilience-modeling/SKILL.md     → npx skills 识别为 "aws-resilience-modeling"
├── chaos-engineering-on-aws/SKILL.md    → npx skills 识别为 "chaos-engineering-on-aws"
└── eks-resilience-checker/SKILL.md      → npx skills 识别为 "eks-resilience-checker" (新增)

安装后:
~/.claude/skills/eks-resilience-checker → symlink → 仓库中的 eks-resilience-checker/
```

#### SKILL.md frontmatter 要求

```yaml
---
name: eks-resilience-checker
description: >
  Assess Amazon EKS cluster resilience against 26 best practice checks
  covering application workloads, control plane, and data plane.
  Outputs structured assessment.json for chaos-engineering-on-aws integration.
  Use when the user wants to evaluate EKS cluster resilience, run resilience
  assessment, check EKS best practices, or prepare for chaos experiments.
  Triggers on: EKS resilience, 韧性评估, cluster assessment, resilience check,
  EKS best practices, 集群评估.
---
```

#### 更新和管理

```bash
# 检查已安装 Skill 是否有更新
npx skills check

# 更新所有 Skill 到最新版
npx skills update

# 列出已安装的 Skill
npx skills list

# 卸载
npx skills remove eks-resilience-checker
```

### 9.2 手动安装（备选）

对于不使用 `npx skills` 的用户：

```bash
# Clone 仓库
git clone https://github.com/panlm/sample-aws-resilience-skill.git

# 手动 symlink 到 Agent skills 目录
ln -s $(pwd)/sample-aws-resilience-skill/eks-resilience-checker ~/.claude/skills/eks-resilience-checker

# 或者直接拷贝
cp -r sample-aws-resilience-skill/eks-resilience-checker ~/.claude/skills/
```

### 9.3 与其他 Skill 仓库的关系

| 仓库 | 内容 | 安装命令 |
|------|------|---------|
| `panlm/sample-aws-resilience-skill` | 4 个韧性 Skill（本仓库） | `npx skills add panlm/sample-aws-resilience-skill --skill '*'` |
| `panlm/skills` | 通用 AWS Skills（research / FIS prepare / execute / log analysis） | `npx skills add panlm/skills --skill '*'` |

两个仓库的 Skill 可以配合使用：
- `panlm/skills` 的 `aws-fis-experiment-prepare` + `aws-fis-experiment-execute` 是独立的 FIS 实验 prepare/execute 工具
- `panlm/sample-aws-resilience-skill` 的 `chaos-engineering-on-aws` 是端到端的混沌工程 Skill（6 步流程）
- 用户可以根据需要选择轻量的 prepare/execute 还是全流程的 chaos skill

| 资料 | 位置 | 用途 |
|------|------|------|
| EKS Resiliency Checkpoints | `eks-resilience-checker/EKS-Resiliency-Checkpoints.md` | 26 项检查详细定义 |
| eks-app-log-analysis | `github.com/panlm/skills/eks-app-log-analysis` | 日志分析模式参考（→ chaos skill 扩展） |
| log_collector.py | `graph-driven-chaos/code/runner/log_collector.py` | 错误分类逻辑参考（→ chaos skill 扩展） |
| chaos-engineering-on-aws PRD | `chaos-engineering-on-aws/doc/prd.md` | Skill PRD 格式参考 |
| fault-catalog.yaml | `chaos-engineering-on-aws/references/fault-catalog.yaml` | FAIL→实验映射的故障类型 |
| AWS EKS Best Practices | `aws.github.io/aws-eks-best-practices` | 检查项理论依据 |
| vercel-labs/skills CLI | `github.com/vercel-labs/skills` | `npx skills add` 分发机制 |
| skills.sh | `skills.sh` | Skill 发现和注册平台 |

---

## 10. 参考实现和执行路径

### 10.1 参考来源

本 Skill 实现参考了以下来源：

| 来源 | 内容 | 如何使用 |
|------|------|---------|
| `awslabs/eks-mcp-server` 未合并 PR | 26 项检查的 Python 实现（6069 行）+ 测试（1328 行） | 参考边界处理逻辑（CRD 不存在、API 差异），直接复制 `eks-resiliency-checks.md` 作为 "Why it matters" 补充文档 |
| `EKS-Resiliency-Checkpoints.md` | 26 项检查的详细定义和合规标准 | 作为 assess.sh 和 SKILL 指令的检查规范 |
| `RadiumGu/Chaosmesh-MCP` | EKS Authentication Methods | 参考设计了三种 kubectl 认证路径（已有 kubeconfig / IAM / ServiceAccount Token） |
| `panlm/skills/eks-app-log-analysis` | 实时/事后双模式日志分析 | 参考设计了 chaos skill 的 log-collector.sh |
| `graph-driven-chaos/log_collector.py` | 5 类错误分类逻辑 | 参考设计了 log-collector.sh 的 classify_line() |

### 10.2 执行路径

```
用户触发 "EKS 韧性评估"
       │
       ▼
┌─ Path A: assess.sh 脚本（推荐）──────────────────────────────┐
│  bash scripts/assess.sh --cluster <name> --region <region>    │
│  纯 bash + kubectl + aws CLI + jq                             │
│  输出 4 个文件：                                              │
│    assessment.json          — 结构化结果 + 实验推荐           │
│    assessment-report.md     — Markdown 报告                   │
│    assessment-report.html   — HTML 报告（颜色编码）           │
│    remediation-commands.sh  — 修复脚本                        │
│  优点：一键完成，结构化输出，可重复，CI/CD 友好               │
└───────────────────────────────────────────────────────────────┘
       │ 如果 assess.sh 不适用
       ▼
┌─ Path B: Agent 逐条执行（默认）──────────────────────────────┐
│  Agent 按 SKILL_EN/ZH.md Step 2 执行 kubectl 命令            │
│  参考 references/check-commands.md                            │
│  优点：Agent 可按上下文跳过检查，灵活性最高                   │
└───────────────────────────────────────────────────────────────┘
```

### 10.3 验证记录

**PetSite 集群实测**（2026-04-04，EKS 1.34，ap-northeast-1）：

| 指标 | 结果 |
|------|------|
| 检查项 | 26（A1-A14 + C1-C5 + D1-D7） |
| PASS | 10 |
| FAIL | 13 |
| INFO | 3 |
| Critical Failures | 5（A2/A4/A5/C4/D3） |
| Compliance Score | 40.0% |
| 实验推荐 | 7 条（A2/A3/A4/A5/A6/A8/D3） |
| 执行时间 | ~90 秒 |
| 输出文件 | 4 个（JSON 42KB, MD 13KB, HTML 22KB, SH 45KB） |

**已修复的运行时问题**：
- `jq --argjson` 在大集群上 `Argument list too long` → 改用 stdin 管道
- A6/A8 的 jq nested reference 需要 `$root` 模式
