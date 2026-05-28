# devops-agent-cn-management — Skills Directory

> 🌐 中文版本: [README.md](./README.md)

This directory contains 10 DevOps Agent Skills designed for **AWS China Region multi-account** scenarios. Upload them to Agent Space to guide the Agent's routing, incident handling, and operational checks in AWS China.

---

## Skill List

| Skill | Purpose | Trigger Examples |
|-------|---------|-----------------|
| **china-region-multi-account-routing** | Routing layer: selects the correct MCP endpoint (aws-cn / aws-cn-2) | "China region", "cn-north-1", "Ningxia", "Beijing" |
| **china-incident-triage** | First response: classify alerts, assess severity, deduplicate | "alert", "incident", "triage", "check this alarm" |
| **china-incident-rca** | Root cause analysis: correlate CloudTrail + deployment events + metric anomalies | "root cause", "why is it down", "deep dive", "investigate" |
| **china-incident-mitigation** | Mitigation guidance: generate CLI commands + require human approval | "how to fix", "fix it", "rollback", "mitigate" |
| **china-account-prevention-checks** | Proactive health checks: single points of failure, quotas, expiring certs, stale credentials | "health check", "proactive", "risk scan" |
| **cn-partition-arn-routing** | Diagnose ARN partition mismatches (`arn:aws:` vs `arn:aws-cn:`) | "AccessDenied", "partition", "wrong ARN" |
| **cross-account-cost-attribution** | Cost comparison across two accounts | "cost", "spending", "which is more expensive", "cost breakdown" |
| **cross-account-inventory-compare** | Resource inventory comparison across accounts | "compare accounts", "drift", "are they consistent" |
| **cross-account-security-posture-check** | Security compliance audit: public S3, MFA, open SG ports, etc. | "security", "compliance", "audit", "any risks" |
| **use-eks-via-call-kubectl** | Guide Agent to use `call_kubectl` (not `use_kubectl`) for EKS queries | "pod status", "kubectl", "check pod" |

---

## Three-Layer Architecture

```
┌─────────────────────────────────────────┐
│            Layer 1: Routing             │
│  china-region-multi-account-routing     │  ← All requests pass through here first
│  cn-partition-arn-routing               │  ← Handles ARN partition issues
│  use-eks-via-call-kubectl               │  ← EKS-specific routing
├─────────────────────────────────────────┤
│         Layer 2: Incident Pipeline      │
│         triage → rca → mitigation       │  ← Sequential flow
├─────────────────────────────────────────┤
│           Layer 3: Operational          │
│  prevention-checks                      │  ← Daily health checks
│  cost-attribution                       │  ← Cost analysis
│  inventory-compare                      │  ← Resource comparison
│  security-posture-check                 │  ← Security audit
└─────────────────────────────────────────┘
```

---

## How to Use

1. Go to **Agent Space → Capabilities → Skills** and upload the corresponding `.zip` file
2. Each directory contains a `SKILL.md`; the Agent auto-triggers the skill via its description field
3. Multiple skills can be loaded simultaneously and work together (e.g., triage can chain into rca)

---

## Notes

- Skills **do not execute commands** — they only guide the Agent's behavior strategy
- All actual API calls are made through the MCP Server's `call_aws` / `call_kubectl` tools
- **Mitigation skill requires explicit human approval** before any write operations are executed
- China region ARNs must use the `arn:aws-cn:` partition — Skills will automatically detect and correct mismatches

---

## Directory Structure

```
devops-agent-cn-management/
├── README.md                          # Chinese version
├── README_EN.md                       # This file (English)
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
