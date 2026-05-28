---
name: cross-account-security-posture-check
description: Run a security and compliance posture audit across both China
  region accounts (aws-cn in cn-northwest-1 and aws-cn-2 in cn-north-1),
  checking for common misconfigurations and unsafe defaults. Use this skill
  when the user asks about security, 安全, 合规, compliance, audit, 审计,
  posture, hardening, baseline, or wants to know if the two China accounts
  are "safe", "following best practices", or "have risks". The audit checks
  public S3 buckets, IAM users missing MFA, root account API activity,
  CloudTrail status, overly-permissive security groups opening sensitive
  ports to 0.0.0.0/0,
  unused/aged IAM access keys, and KMS key rotation. Report findings grouped
  by severity (critical / high / medium / low) and by account.
---

# Cross-Account Security Posture Check

Routing behavior is governed by `china-region-multi-account-routing`.
This skill assumes the agent will hit both China MCPs and focuses on
**what to check, how to classify findings, and how to report them**.

## When to use

Use this skill when the user's intent is security/compliance review
covering the two China accounts. Examples:

- "两个中国区账号合规情况怎么样"
- "帮我看看两个账号有没有公开的 S3"
- "哪些 IAM 用户没开 MFA"
- "两个账号 CloudTrail 开了没"
- "找一下 0.0.0.0/0 的 SG"
- "security audit on both China accounts"

Do **not** use this skill for:
- Active remediation (this skill is read-only — findings only)
- Penetration testing or exploit attempts
- Compliance framework attestation (PCI, SOC2, etc. — different scope)

## Check catalog

Run these checks **in parallel across both accounts**. Each check has a
severity classification the agent should preserve in the report.

| # | Check | API call                                          | Severity if found |
|---|-------|---------------------------------------------------|-------------------|
| 1 | Public S3 buckets | `s3api list-buckets` + `get-bucket-policy-status` + `get-public-access-block` per bucket | **Critical** |
| 2 | Root account API usage (last 30 days) | `cloudtrail lookup-events --lookup-attributes AttributeKey=Username,AttributeValue=Root` | **Critical** |
| 3 | IAM users without MFA | `iam list-users` + `iam list-mfa-devices` per user | **High** |
| 4 | CloudTrail not enabled in current region | `cloudtrail describe-trails` + `get-trail-status` | **High** |
| 5 | Security groups allowing `0.0.0.0/0` on sensitive ports (22, 3389, 3306, 5432, 6379, 27017) | `ec2 describe-security-groups` + inspect `IpPermissions` | **High** |
| 6 | IAM access keys older than 90 days | `iam list-users` + `iam list-access-keys` + check `CreateDate` | **Medium** |
| 7 | IAM access keys unused for 90+ days | `iam get-access-key-last-used` per key | **Medium** |
| 8 | KMS customer-managed keys without rotation | `kms list-keys` + `kms get-key-rotation-status` | **Low** |
| 9 | Default VPC still exists with default SG allowing all | `ec2 describe-vpcs --filters Name=isDefault,Values=true` + SG inspection | **Low** |

If the user narrows the ask ("只看 S3", "只看 IAM"), run only the
matching checks. If the user says "all" / "full audit" / "全量", run all.

## Procedure

### Step 1 — Scope

Parse which checks to run (default: all 9). Parse which accounts to cover
(default: both; `china-region-multi-account-routing` handles the choice).

### Step 2 — Parallel execution

For each (account, check) pair, issue the required API calls. Prefer
wide parallelism — a full audit of 2 accounts × 9 checks should complete
in one round-trip wave, not 18.

### Step 3 — Classify findings

Each finding is a 5-tuple:

```
(account, severity, check_id, resource_id, evidence)
```

Examples:

```
(aws-cn,   Critical, public-s3,   my-prod-logs,     "ACL allows public read")
(aws-cn-2, High,     iam-mfa,     user/ci-deployer, "No MFA device configured")
(aws-cn,   High,     sg-open,     sg-0abc123,       "0.0.0.0/0 → port 22")
```

### Step 4 — Report

Two-level grouping: **by severity first, then by account within severity**.
This makes the critical stuff impossible to miss even if the list is long.

```
## 🔴 Critical (2 findings)

### aws-cn (Ningxia)
- [public-s3] Bucket `my-prod-logs` allows public read
  - Evidence: `get-bucket-acl` returned grantee `AllUsers`

### aws-cn-2 (Beijing)
- [root-usage] Root account issued 3 API calls in the last 30 days
  - Events: ConsoleLogin at 2026-04-22, CreateAccessKey at 2026-04-22

## 🟠 High (N findings)

### aws-cn (Ningxia)
- [iam-mfa] 2 IAM users without MFA: `alice`, `ci-deployer`
- [sg-open] 1 SG open to world on port 22: `sg-0abc123` in vpc-0def

### aws-cn-2 (Beijing)
- [cloudtrail] CloudTrail not enabled in cn-north-1

## 🟡 Medium (...)
## 🟢 Low (...)
```

After the findings, include a one-line scorecard:

```
aws-cn:    2 Critical / 3 High / 4 Medium / 1 Low
aws-cn-2:  1 Critical / 2 High / 1 Medium / 0 Low
```

### Step 5 — Suggest next action (not remediation)

End with a single sentence pointing at the highest-severity finding:

> "Recommend addressing `public-s3: my-prod-logs` first — a public bucket
> is immediately exploitable. Want me to show you the exact remediation
> CLI command (without running it)?"

This invites a follow-up but does not auto-remediate.

## Things not to do

- **Do not** execute remediation actions (e.g., `put-public-access-block`,
  `delete-access-key`, `revoke-security-group-ingress`). This skill is
  **read-only**. The user must explicitly approve any write.
- **Do not** fail the entire audit if one check errors on one account.
  Report the check as `[error: <message>]` for that account and continue
  with the rest.
- **Do not** collapse findings across accounts — `public-s3 in aws-cn` and
  `public-s3 in aws-cn-2` are two separate findings, even if they have the
  same check_id.
- **Do not** assume an IAM user without MFA is necessarily a violation —
  service accounts sometimes deliberately skip MFA. Report it, do not
  judge it.
- **Do not** rely on AWS Config or Security Hub being enabled — both are
  available in China but may not be configured. This skill's checks are
  all on primitive AWS APIs that work without extra setup.

## Examples

**Input**: "两个中国区账号有没有公开的 S3"

**Action**: Check #1 only, both accounts in parallel. Report findings
grouped by account. If none, say "No public S3 buckets found in either
account (checked N buckets in aws-cn, M in aws-cn-2)."

**Input**: "full security audit on both China accounts"

**Action**: All 9 checks, both accounts, full report as shown in Step 4.

**Input**: "北京账号 IAM 有没有问题"

**Action**: Checks #3, #6, #7 (IAM-related) on `aws-cn-2` only. Report
grouped by severity. Do not touch aws-cn.

**Input**: "两个账号哪个安全做得好"

**Action**: Run full audit on both, return the scorecard summary first,
then let the user drill down. Explicitly state that "fewer findings" is a
rough proxy only — a small account naturally has fewer findings than a
large one.
