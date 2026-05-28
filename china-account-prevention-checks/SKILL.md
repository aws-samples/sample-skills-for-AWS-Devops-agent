---
name: china-account-prevention-checks
description: Proactive prevention and pre-alarm health checks across the two
  China region accounts (aws-cn and aws-cn-2). Use this skill when the user
  asks about prevention, 防护, 预防, proactive, 体检, health check, risk
  assessment, 潜在风险, 隐患, or "what might break soon", and when the
  Evaluation agent runs scheduled recommendation workflows. Looks for
  conditions that predict future incidents — single points of failure,
  service quotas nearing limits, stale AMIs, aging credentials, certificates
  expiring within 30 days, deprecated Lambda runtimes. This skill is
  distinct from cross-account-security-posture-check, which reports
  current-state security risk. Prevention predicts future failure; security
  posture describes current exposure.
---

# China Account Prevention Checks

Routing is governed by `china-region-multi-account-routing`. This skill
runs predictive checks across both accounts and ranks findings by
**time-to-likely-incident**, not by static severity.

## Intended agent type

Upload with Agent Type **Evaluation** selected. This skill loads during
scheduled prevention workflows and when a user asks for proactive review.
Do not leave Agent Type as Generic — it is noisy in incident paths.

## When to use

- Scheduled Evaluation runs (daily/weekly)
- User phrases like "体检一下两个账号", "有什么潜在风险", "哪些东西快挂",
  "proactive review", "preventive maintenance"
- Before major events (holiday freeze, product launch, customer demo)

Do **not** use during an active incident — the user wants triage/RCA then,
not a list of unrelated future risks.

## Check catalog

Runs across both accounts in parallel. Each check has a `time-to-incident`
estimate that ranks the finding.

| # | Check | API | Time-to-incident |
|---|-------|-----|------------------|
| 1 | RDS `MultiAZ=false` on prod-tagged DBs | `rds describe-db-instances` | ~30 days (AZ event MTBF) |
| 2 | Auto Scaling groups with `min=desired=max=1` on prod | `autoscaling describe-auto-scaling-groups` | ~60 days (instance-level MTBF) |
| 3 | Service quotas above 80% used | `service-quotas get-service-quota` + CloudWatch usage metrics | Varies — if growth rate suggests hit in 30 days, escalate |
| 4 | EC2 running AMIs older than 180 days | `ec2 describe-instances` + `describe-images` | ~90 days (CVE exposure compounds) |
| 5 | IAM access keys older than 60 days (predictive of expiry) | `iam list-access-keys` + age check | 30 days until team's 90-day rotation deadline |
| 6 | ACM certs expiring within 30 days | `acm list-certificates` + `describe-certificate` | **IMMEDIATE if < 14 days** |
| 7 | Lambda runtimes marked deprecated by AWS | `lambda list-functions` + runtime vs. [deprecation schedule] | Varies — check official AWS deprecation dates |
| 8 | EKS node groups with `desiredSize=1` | `eks describe-nodegroup` | ~60 days |
| 9 | Single-replica Kubernetes Deployments in critical namespaces | via kubectl MCP if available, else skip | ~60 days |

If the user narrows the ask ("只看证书", "RDS 有没有问题"), run only the
relevant checks. Default = all checks.

## Procedure

### Step 1 — Parallel execution

For each (account, check) pair, issue API calls concurrently. Do not
serialize across the 9 checks × 2 accounts.

### Step 2 — Classify by urgency

Assign each finding one of four urgency bands based on time-to-incident:

- **IMMEDIATE** (< 14 days until failure)
- **SOON** (14–30 days)
- **WATCH** (30–60 days)
- **TRACK** (60–90 days or longer)

### Step 3 — Rank and present

Two-level grouping: **urgency first, then account within urgency**.

```
## 🔴 IMMEDIATE (< 14 days)

### aws-cn (Ningxia)
- [acm-expiry] Certificate `prod.yingchu.cloud` expires in 7 days (2026-05-18)
  → Trigger renewal; DNS validation CNAME already present

## 🟠 SOON (14–30 days)

### aws-cn (Ningxia)
- [iam-key-age] User `ci-deployer` access key created 62 days ago — team
  policy is 90 days, rotation window opens in 28 days

### aws-cn-2 (Beijing)
- [rds-multiaz] Prod RDS `prod-db-nrt` has MultiAZ disabled; AZ outage
  would cause ~30 min downtime

## 🟡 WATCH (30–60 days) — N findings omitted (ask to expand)
## 🟢 TRACK (60+ days) — M findings omitted
```

### Step 4 — Scorecard + nudge

End with a per-account scorecard:

```
aws-cn:    1 Immediate / 1 Soon / 3 Watch / 2 Track
aws-cn-2:  0 Immediate / 1 Soon / 2 Watch / 1 Track
```

And a single actionable nudge pointing at the highest-urgency item.

## Things not to do

- **Do not** execute remediation (rotate keys, renew certs, scale up).
  This skill is read-only.
- **Do not** duplicate findings already reported by
  `cross-account-security-posture-check` in the same session. If the user
  already ran security audit, defer overlapping checks to it.
- **Do not** raise alarms on AWS-managed auto-renewing resources (e.g.,
  ACM certs issued via AWS have auto-renewal when DNS validation record is
  present — only flag when renewal is blocked).
- **Do not** assume service quotas in China mirror global region quotas;
  they can differ. Always read the live quota, don't hardcode.
- **Do not** confuse this with security-posture. If in doubt — "is this
  currently exploitable?" → security. "Will this fail sometime?" → prevention.

## Examples

**Input**: "体检一下两个中国区账号"

**Action**: Run all 9 checks on both accounts, produce the full urgency-
banded report + scorecard.

**Input**: "两个账号证书还有多久过期"

**Action**: Check #6 only, both accounts, list all certs with expiry date
and days remaining.

**Input**: "哪些是 immediate 要处理的"

**Action**: Same as full run but filter to IMMEDIATE band only. If empty,
confirm "No immediate-urgency findings in either account."
