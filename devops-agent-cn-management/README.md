# devops-agent-cn-management — Skills 目录

> 🌐 English version: [README_EN.md](./README_EN.md)

本目录包含 10 个面向 **AWS 中国区多账号** 场景的 DevOps Agent Skills，上传到 Agent Space 后指导 Agent 在中国区正确路由、处理事件、执行巡检。

---

## Skill 列表

| Skill | 用途 | 触发词示例 |
|-------|------|-----------|
| **china-region-multi-account-routing** | 路由层：决定用哪个 MCP endpoint（aws-cn / aws-cn-2） | "中国区", "cn-north-1", "宁夏", "北京" |
| **china-incident-triage** | 告警首响：分类、判断严重性、去重 | "告警", "出事了", "triage", "看一下这个告警" |
| **china-incident-rca** | 根因分析：CloudTrail + 部署事件 + metric 异常关联 | "根因", "为什么挂", "deep dive", "调查" |
| **china-incident-mitigation** | 缓解建议：输出 CLI 命令 + 人工审批 | "怎么修", "修复", "rollback", "缓解" |
| **china-account-prevention-checks** | 预防性巡检：单点故障、配额、证书过期、老化凭证 | "体检", "预防", "隐患", "proactive" |
| **cn-partition-arn-routing** | ARN partition 误写诊断（`arn:aws:` vs `arn:aws-cn:`） | "AccessDenied", "partition", "ARN 不对" |
| **cross-account-cost-attribution** | 双账号成本对比 | "花费", "成本", "哪个贵", "cost breakdown" |
| **cross-account-inventory-compare** | 双账号资源清单对比 | "对比两个账号", "drift", "一致吗" |
| **cross-account-security-posture-check** | 安全合规审计：公开 S3、MFA、SG 开放端口等 | "安全", "合规", "audit", "有风险吗" |
| **use-eks-via-call-kubectl** | 引导 Agent 用 `call_kubectl`（而非 `use_kubectl`）查 EKS | "pod status", "kubectl", "查pod" |

---

## 三层架构

```
┌─────────────────────────────────────────┐
│            Layer 1: Routing             │
│  china-region-multi-account-routing     │  ← 所有请求先过这层，决定打哪个 MCP
│  cn-partition-arn-routing               │  ← ARN 特殊问题走这个
│  use-eks-via-call-kubectl               │  ← kubectl 特殊路由
├─────────────────────────────────────────┤
│         Layer 2: Incident Pipeline      │
│         triage → rca → mitigation       │  ← 按顺序流转
├─────────────────────────────────────────┤
│           Layer 3: Operational          │
│  prevention-checks                      │  ← 日常巡检
│  cost-attribution                       │  ← 成本分析
│  inventory-compare                      │  ← 资源对比
│  security-posture-check                 │  ← 安全审计
└─────────────────────────────────────────┘
```

---

## 使用方式

1. 在 **Agent Space → Capabilities → Skills** 上传对应的 `.zip` 文件
2. 每个目录下含一个 `SKILL.md`，Agent 通过 description 字段自动触发对应 skill
3. 多个 skill 可同时加载，彼此协作（如 triage 触发后可链式调用 rca）

---

## 注意事项

- Skills **不执行命令**，只引导 Agent 的行为策略
- 所有实际 API 调用通过 MCP Server 的 `call_aws` / `call_kubectl` 工具完成
- **Mitigation skill 强制要求人工审批**，不会自动执行写操作
- 中国区 ARN 必须使用 `arn:aws-cn:` partition，Skills 会自动检测并纠正

---

## 目录结构

```
devops-agent-cn-management/
├── README.md                          # 本文件（中文）
├── README_EN.md                       # English version
├── china-region-multi-account-routing/
│   └── SKILL.md
├── china-incident-triage/
│   └── SKILL.md
├── china-incident-rca/
│   └── SKILL.md
├── china-incident-mitigation/
│   └── SKILL.md
├── china-account-prevention-checks/
│   └── SKILL.md
├── cn-partition-arn-routing/
│   └── SKILL.md
├── cross-account-cost-attribution/
│   └── SKILL.md
├── cross-account-inventory-compare/
│   └── SKILL.md
├── cross-account-security-posture-check/
│   └── SKILL.md
└── use-eks-via-call-kubectl/
    └── SKILL.md
```
