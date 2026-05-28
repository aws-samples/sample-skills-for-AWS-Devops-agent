---
name: cross-account-cost-attribution
description: Retrieve, compare, and attribute AWS spend across the two China
  region accounts (aws-cn in cn-northwest-1 and aws-cn-2 in cn-north-1).
  Use this skill when the user asks about cost, spend, billing, 花费, 成本,
  账单, 多少钱, expensive, 贵, top services, cost breakdown, month-over-month,
  budget, or wants to know which China account spends more and on what. Covers
  month-to-date, last month, last 90 days, and custom time ranges. Also use
  when the user wants to correlate cost with specific resources or services
  (e.g. "哪个账号的 EC2 花费高"). Report totals per account, top services
  per account, and deltas vs previous period when available.
---

# Cross-Account Cost Attribution

Routing behavior is governed by `china-region-multi-account-routing`.
This skill assumes the agent is already prepared to hit both China MCPs
and focuses on **how to query cost data and present a comparative view**.

## When to use

Use this skill whenever the user's intent involves monetary comparison or
attribution across the two China accounts. Examples:

- "这个月两个账号各花了多少"
- "哪个账号贵"
- "两个账号的 EC2 花费对比"
- "北京账号上个月比这个月涨了多少"
- "top 5 services by cost in each account"
- "我这两个中国区账号一共 MTD 多少"

Do **not** use this skill for:
- Single-account cost queries (the user already specifies account → let
  the base `aws-api-mcp-server` handle it)
- AWS global cost (this skill is China-partition only)
- Forecasting / budget alerts (out of scope — different API surface)

## China partition cost API note

AWS China has its own Billing/Cost Explorer API surface at
`bcm-data-exports.cn-northwest-1.amazonaws.com.cn` and
`ce.cn-north-1.amazonaws.com.cn`. The CLI commands are the same as global
(`aws ce get-cost-and-usage`), but credentials must be the China-partition
AK/SK — which is exactly what each MCP pod already has. Do not use
`aws-global` credentials; they will fail with `AuthFailure`.

## Procedure

### Step 1 — Determine the time range

Parse from the user's query. Default if unspecified: **month-to-date (MTD)
for the current month**. Common mappings:

| User phrase | Time range |
|---|---|
| "这个月" / "this month" / "MTD" | First day of current month → today |
| "上个月" / "last month" | First day → last day of previous month |
| "过去 30 天" / "last 30 days" | Today − 30 days → today |
| "今年" / "YTD" | Jan 1 of current year → today |
| "环比" / "month-over-month" | Return both current MTD and previous full month |

Always include the exact date range in the response so the user can verify.

### Step 2 — Query Cost Explorer on both endpoints in parallel

Use `aws ce get-cost-and-usage` with:
- `--time-period Start=...,End=...`
- `--granularity MONTHLY` (or `DAILY` if range ≤ 31 days and user wants
  trend)
- `--metrics UnblendedCost`
- `--group-by Type=DIMENSION,Key=SERVICE` (for top-services breakdown)

Call on both `aws-cn` and `aws-cn-2` MCPs concurrently.

### Step 3 — Aggregate and compare

For each account, compute:
1. **Total spend** (sum of `UnblendedCost.Amount` across groups)
2. **Top N services** (default N=5, sorted by amount desc)
3. **Delta vs prior period** (if the user asked for month-over-month or
   if the default 1-month view makes comparison obvious)

### Step 4 — Present

Two-part format. Headline numbers first, then breakdown tables.

**Headline**:

```
Date range: 2026-05-01 → 2026-05-11 (MTD, 11 days)

aws-cn   (Ningxia): ¥ 1,234.56  (ΔMoM: +8% vs April)
aws-cn-2 (Beijing): ¥   456.78  (ΔMoM: −3% vs April)
Combined:           ¥ 1,691.34
```

**Top-services table** (side-by-side):

| Service         | aws-cn | aws-cn-2 |
|-----------------|-------:|---------:|
| Amazon EC2      |  ¥ 800 |   ¥ 200  |
| Amazon RDS      |  ¥ 300 |   ¥ 150  |
| Amazon S3       |  ¥  80 |   ¥  50  |
| Data Transfer   |  ¥  30 |   ¥  30  |
| Other           |  ¥  24 |   ¥  26  |
| **Total**       | **¥ 1,234.56** | **¥ 456.78** |

### Step 5 — Interpret (1–3 sentences)

State one observation: where is the bigger account spending, is the delta
normal, is there an outlier service. Do not recommend cost optimization
unless asked — that requires deeper data than this skill retrieves.

## Things not to do

- **Do not** convert currencies. China region returns `CNY`; present as
  `¥` and do not add a `$` equivalent. Exchange rate assumptions cause
  confusion and you do not have a rate source.
- **Do not** compare different time ranges across the two accounts. If
  one account's data is incomplete (e.g. CE has 24h lag), note the
  discrepancy explicitly instead of silently aligning.
- **Do not** use `--metrics BlendedCost` or `AmortizedCost` without the
  user asking. Default to `UnblendedCost` — it matches what users see on
  the billing console.
- **Do not** claim a spending anomaly unless you have ≥2 prior periods
  for comparison. One-month deltas on new accounts are noisy.
- **Do not** include detailed SKU-level breakdown unless the user asks.
  Service-level is the default granularity.

## Examples

**Input**: "这两个账号这个月花了多少"

**Action**: MTD on both in parallel, service-level breakdown, side-by-side
table. Report total per account + combined.

**Input**: "哪个账号贵，贵在哪"

**Action**: Current MTD on both, find the higher one, break down its top-3
services, highlight which service drives the difference.

**Input**: "北京账号 EC2 比上月涨了没"

**Action**: `aws-cn-2` only, current MTD vs previous full month, filter
service=`Amazon EC2`. Report delta in absolute ¥ and percent. Note if the
current month is partial (MTD) and adjust interpretation accordingly.

**Input**: "过去 90 天两个账号哪天花的最多"

**Action**: Both accounts, daily granularity, past 90 days. Find the
peak-spend day per account and report with date + amount + top service
that day.
