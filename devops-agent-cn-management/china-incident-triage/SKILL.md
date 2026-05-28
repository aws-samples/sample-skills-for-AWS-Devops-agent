---
name: china-incident-triage
description: First-response triage for an incoming alarm, ticket, or failure
  report originating from either China account (aws-cn or aws-cn-2). Use
  this skill when the trigger is an alarm name, CloudWatch alarm payload,
  SIM ticket body, error log snippet, or a user phrase such as 告警, 出事了,
  服务挂了, incident, triage, 分类, 初步判断, 看一下这个告警, what happened.
  Determines which of the two accounts is affected, classifies the incident
  into one of six classes (compute / network / identity-credentials / data
  / cost / unknown), estimates severity from the signal, and checks whether
  a similar incident fired recently so duplicates are marked. Output is a
  short triage card that hands off to RCA or mitigation depending on
  severity. This skill is the entry point of the incident response pipeline.
---

# China Incident Triage

Routing is governed by `china-region-multi-account-routing`. This skill
assumes an incident signal has just arrived and needs initial
classification in < 60 seconds of wall time.

## Intended agent type

Upload with Agent Type **Incident Triage** selected.

## When to use

- A CloudWatch alarm payload arrives
- A SIM ticket autocuts to the resolver group
- A user pastes an error log or says "XXX 挂了" / "something is broken"
- User phrases: 告警, 出事了, incident, triage, 初步判断, 看下这个, what's wrong

Do **not** use this skill for known-classified incidents (user already
said "this is a network issue, investigate" → skip triage, go to RCA).

## Output contract — the Triage Card

Every invocation produces a **Triage Card** with exactly these fields.
Anything longer dilutes triage speed.

```
╭─ Triage Card ─────────────────────────────────────────────────────────╮
│ Account:     aws-cn  (Ningxia, cn-northwest-1)                        │
│ Class:       Network                                                  │
│ Severity:    SEV-2 (customer-facing impact probable)                  │
│ Resource:    arn:aws-cn:elasticloadbalancing:.../app/prod-alb/...     │
│ First seen:  2026-05-11 14:22 CST                                     │
│ Duplicate?   No (no similar alarm in last 24h)                        │
│ Next step:   Hand off to china-incident-rca with this card            │
╰───────────────────────────────────────────────────────────────────────╯
```

## Procedure

### Step 1 — Determine affected account

From the signal, extract account attribution in this order (first match wins):

1. **ARN in signal** — check ARN partition (`arn:aws-cn:...`) and 12-digit
   account ID segment
2. **Region in signal** — `cn-northwest-1` → aws-cn, `cn-north-1` → aws-cn-2
3. **Resource tag** (if the signal includes tags like `Account=aws-cn`)
4. **Host / endpoint hostname** — `aws-cn.yingchu.cloud` vs `aws-cn-2.yingchu.cloud`
5. **Ask the user** if none of the above resolves

### Step 2 — Classify the incident

Bucket the signal into **exactly one** class:

| Class | Typical signals |
|-------|-----------------|
| **Compute** | EC2 status check failed, ASG unable to launch, EKS pod crashloop, Lambda throttled, ECS task exits |
| **Network** | ALB 5xx spike, target unhealthy, NAT gateway errors, VPC Lattice disconnect, DNS resolution failure, Route 53 health check failed |
| **Identity/Credentials** | `AuthFailure`, `ExpiredToken`, `InvalidClientTokenId`, `SignatureDoesNotMatch`, IAM policy eval errors, STS AssumeRole failures |
| **Data** | RDS connection errors, DynamoDB throttle, S3 4xx/5xx, OpenSearch red status, replication lag |
| **Cost** | Budget alarm, anomaly detection trigger, quota-based alarm (API throttling from quota) |
| **Unknown** | Signal is ambiguous — log a note and request user clarification |

If the signal spans multiple classes (e.g., compute failure caused by
credentials), pick the **root-evident** class and note the secondary.

### Step 3 — Estimate severity

Use a simple rubric:

| Severity | Criterion |
|----------|-----------|
| **SEV-1** | Complete account outage, multiple services down, data loss in progress |
| **SEV-2** | Single service customer-facing impact, 5xx rate > 5%, MCP endpoint down |
| **SEV-3** | Degraded latency, single-AZ impact, partial feature failure |
| **SEV-4** | Monitoring-only, internal tool affected, no customer impact |
| **SEV-5** | Informational, metric threshold crossed without user-visible effect |

Default to **SEV-3** when the signal is ambiguous — escalate via RCA
findings later if needed.

### Step 4 — Dedup check

Before handing off, query:

```
aws cloudwatch describe-alarms --alarm-name-prefix <prefix>
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=<event> \
  --max-items 20 --start-time <24h ago>
```

If a very similar alarm fired within the last 24 hours and is still active
or recently resolved, mark **Duplicate? Yes** with the original incident
ID. This prevents RCA duplication.

### Step 5 — Hand off

Emit the Triage Card. Based on severity:

- **SEV-1 / SEV-2** → "Escalating to RCA; will auto-invoke china-incident-rca."
- **SEV-3** → "Triage complete. Run RCA when ready."
- **SEV-4 / SEV-5** → "Low impact; log and monitor. Skip RCA unless trend develops."

## Things not to do

- **Do not** start RCA investigation during triage. Triage is < 60 seconds
  of classification, not root-cause discovery.
- **Do not** speculate on cause. "Probably network" in the Class field is
  fine; "probably because someone changed the SG last night" is RCA's job.
- **Do not** escalate to SEV-1 based on noisy signals alone. Check for
  corroborating indicators (multiple metrics, multiple alarm targets)
  before claiming account-wide outage.
- **Do not** classify into Unknown without first checking account, region,
  and service fields in the signal. Unknown is a last resort.
- **Do not** skip the dedup check. Repeated investigations on a flapping
  alarm waste agent time and user attention.

## Examples

**Input**: CloudWatch alarm `prod-alb-5xx-rate` state=ALARM in cn-northwest-1

**Action**:
```
Account:     aws-cn (cn-northwest-1)
Class:       Network
Severity:    SEV-2
Resource:    prod-alb (ALB)
First seen:  <alarm.StateChangeTime>
Duplicate?   <check last 24h>
Next step:   Hand off to china-incident-rca
```

**Input**: "aws-cn-2 Lambda throwing AuthFailure since 14:00"

**Action**:
```
Account:     aws-cn-2 (cn-north-1)
Class:       Identity/Credentials
Severity:    SEV-2 (likely wide-impact — credentials power all calls)
Resource:    <Lambda function ARN if provided, else "Lambda fleet">
First seen:  ~14:00 today
Duplicate?   Check recent AuthFailure rate
Next step:   Hand off to china-incident-rca (prioritize: check recent
             Secrets Manager update, check IAM key age)
```

**Input**: "中国区好像不太对劲"

**Action**: Signal too vague for triage. Ask: "Which account, aws-cn or
aws-cn-2? And what symptom — slow response, error, or metric anomaly?"
Do not emit a Triage Card on speculation.
