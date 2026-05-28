---
name: cross-account-inventory-compare
description: Query the same AWS resource type (EC2, RDS, VPC, subnets, security
  groups, S3 buckets, Lambda functions, IAM roles, CloudFormation stacks, etc.)
  across both China region MCPs (aws-cn and aws-cn-2) and present a side-by-side
  comparison. Use this skill when the user asks to compare, diff, list across
  both, find differences, find matching resources by name or tag, or detect
  drift/inconsistency between the two China accounts. Triggers on phrases like
  "对比两个账号", "两边", "两个中国区", "aws-cn vs aws-cn-2", "宁夏和北京",
  "compare the two China accounts", "diff", "drift", "同名资源", "一致吗".
  Also use when the user wants an inventory that must cover both accounts
  regardless of which.
---

# Cross-Account Resource Inventory & Comparison

Routing behavior (endpoint ↔ region mapping, disambiguation rules, output
labeling) is governed by the `china-region-multi-account-routing` skill;
this skill assumes that routing is already in effect and focuses on
**how to query, correlate, and diff resources across both accounts**.

## When to use

Use this skill whenever the user's intent requires data from both China
region accounts and a comparative or unified view of that data. Examples:

- "对比两个账号的 VPC CIDR"
- "两个中国区账号各自有几个 EC2 在跑"
- "北京账号有没有和宁夏同名的 security group"
- "哪些 S3 bucket 只在一个账号里存在"
- "两边的 IAM role 一致吗"

Do **not** use this skill when the user asks about a single explicitly-named
account ("查宁夏的 EC2") — that is single-account inventory, not comparison.

## Procedure

### Step 1 — Identify the resource type and scope

Parse the user's request to extract:
- **Resource type** (EC2 instance, RDS DB, VPC, S3 bucket, IAM role, …)
- **Scope** (全部 / 运行中 / 某 tag / 某 name prefix / 某 region)
- **Comparison key** (name, tag `Name`, ID, ARN, configuration field)

If any of these is ambiguous, ask one clarifying question before proceeding.

### Step 2 — Run the equivalent describe/list on both endpoints in parallel

Call the same AWS API on both `aws-cn` and `aws-cn-2` MCP endpoints
concurrently (not sequentially). Use the narrowest filter the user's
request allows — avoid listing all resources of a type if a tag or name
filter works.

Examples:
- EC2: `aws ec2 describe-instances --filters Name=instance-state-name,Values=running`
- RDS: `aws rds describe-db-instances`
- VPC: `aws ec2 describe-vpcs`
- S3:  `aws s3api list-buckets` (plus `get-bucket-location` per bucket
       since S3 is regional in China)
- IAM: `aws iam list-roles` (partition-wide, same per account)

### Step 3 — Correlate across accounts

Match resources across the two accounts using the comparison key from
Step 1. Partition the results into three buckets:

1. **Only in aws-cn** — present in Ningxia, absent in Beijing
2. **Only in aws-cn-2** — present in Beijing, absent in Ningxia
3. **In both** — present in both accounts (matched by key)

For bucket 3, surface any **configuration differences** the user asked
about (e.g., different CIDRs, different instance types, different tags).

### Step 4 — Present the diff

Use a table when comparing configuration:

| Resource (by name) | aws-cn (Ningxia) | aws-cn-2 (Beijing) |
|--------------------|------------------|--------------------|
| `prod-web-sg`      | `0.0.0.0/0:443` allowed | `0.0.0.0/0:443` allowed |
| `prod-db-sg`       | exists            | **missing**        |

Use grouped sections when listing disjoint sets:

```
### Only in aws-cn (Ningxia)
- i-0aaa... (t3.medium)
- i-0bbb... (t3.small)

### Only in aws-cn-2 (Beijing)
- i-0xxx... (t3.large)

### In both
(none, or list matched pairs)
```

### Step 5 — Summarize findings

End the response with 1–3 sentences of interpretation: is the drift
expected (e.g. DR-style asymmetry) or unexpected (e.g. missing resource)?
Do not remediate — this skill is read-only analysis.

## Things not to do

- **Do not** assume two resources with the same name are "the same" — they
  are different AWS resources in different accounts. Say "matched by name"
  or "matched by tag X", never "the same resource".
- **Do not** silently drop resources from one side if the API fails on
  that side. If `aws-cn-2` returned an error, report "aws-cn-2: failed
  with <error>" and still show aws-cn's data — do not pretend aws-cn-2
  has no resources.
- **Do not** iterate sequentially on the two accounts when parallel is
  possible. Users expect one round-trip latency, not two.
- **Do not** reformat IDs or ARNs when presenting — they need to be
  copy-pasteable back into AWS CLI.

## Examples

**Input**: "对比两个账号的 VPC CIDR"

**Action**: Parallel `aws ec2 describe-vpcs` on both. Extract `CidrBlock`
and `Tags[Name]` per VPC. Match by tag `Name`. Output:

| VPC Name  | aws-cn (Ningxia) | aws-cn-2 (Beijing) |
|-----------|------------------|--------------------|
| `default` | `172.31.0.0/16`  | `172.31.0.0/16`    |
| `prod`    | `10.0.0.0/16`    | `10.1.0.0/16`      |
| `dev`     | `10.10.0.0/16`   | — (absent)         |

Summary: "aws-cn-2 is missing the `dev` VPC; `default` overlaps (expected,
both accounts use AWS's default). `prod` CIDRs are distinct (no overlap,
safe to peer later if needed)."

**Input**: "两个账号各自多少个 running EC2"

**Action**: Parallel `describe-instances --filters state=running`. Output
just counts plus instance IDs:

```
aws-cn (Ningxia):   3 running — i-0a, i-0b, i-0c
aws-cn-2 (Beijing): 1 running — i-0x
```

**Input**: "哪些 IAM role 只存在于宁夏账号，北京账号没有"

**Action**: Parallel `aws iam list-roles` on both. Compute set difference.
Output:

```
### Only in aws-cn (Ningxia) — 2 roles
- mcp-pod-reader
- eks-node-group-extra

### Only in aws-cn-2 (Beijing) — 0 roles

### In both — 14 roles
(not listed unless requested)
```
