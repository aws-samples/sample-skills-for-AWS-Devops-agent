---
name: china-incident-rca
description: Root cause analysis for a triaged incident in either China
  account (aws-cn or aws-cn-2). Use this skill after triage has produced a
  Triage Card, or when the user directly asks RCA, 根本原因, 根因, 为什么挂,
  why did X fail, deep dive, deep investigation, 深入分析, dig into, 调查.
  Correlates the CloudTrail API log window around the incident, recent
  deploy events (CloudFormation stack events, CodeDeploy, ECR pushes,
  Lambda updates), metric anomalies against prior-week baseline, and
  cross-account blast radius — specifically, whether the same failure
  pattern also hit the other China account around the same time, which
  would suggest a shared upstream cause (IAM partition-wide, AWS region
  event, or common dependency). Produces a single root-cause hypothesis
  plus the evidence chain. Does NOT execute remediation.
---

# China Incident RCA

Routing is governed by `china-region-multi-account-routing`. Triage
hand-off is governed by `china-incident-triage`. This skill picks up from
a Triage Card and produces a **single root-cause hypothesis with
evidence**.

## Intended agent type

Upload with Agent Type **Incident RCA** selected.

## When to use

- A Triage Card exists and severity is SEV-3 or higher
- User asks RCA-style questions: "根本原因", "why did X fail", "深入分析"
- A recurring alarm needs explanation even if low-severity

Do **not** use this skill for:
- Ambiguous or untriaged incidents (run triage first)
- "How do I fix this?" questions (use mitigation skill)
- Capacity or cost analysis (different skills)

## Investigation framework — the 4 axes

Check these four axes **in parallel**. Most root causes surface on
exactly one axis; the others serve as confirmation or elimination.

### Axis 1 — CloudTrail API log

For the 30-minute window around `incident_start_time` (15 min before,
15 min after), in the affected account's region:

```
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=ResourceName,AttributeValue=<resource> \
  --start-time <incident_start - 15min> \
  --end-time <incident_start + 15min>
```

Look for:
- Who (principal) made changes?
- What was changed (Modify*, Delete*, Update*, Put*)?
- Was it human (IAM user) or automated (role from CI/pipeline)?
- Was this change correlated with the alarm start time within 5 minutes?

### Axis 2 — Deploy correlation

Query in parallel:

- `aws cloudformation describe-stack-events --stack-name <stack>` (if resource is CFN-managed)
- `aws codedeploy list-deployments --deployment-group-name <group>` (if applicable)
- `aws ecr describe-images --repository-name <repo>` (image push times)
- `aws lambda list-versions-by-function --function-name <fn>` (Lambda updates)
- Git push events from the pipeline (if GitFarm/GitHub integration active)

A deploy within **±30 minutes** of incident start is a top suspect.

### Axis 3 — Metric anomaly vs. baseline

Pull two `GetMetricData` windows for the affected resource:

- **Current**: `incident_start - 1h` to `incident_start + 30min`
- **Baseline**: same 90-minute window, 7 days earlier

Compute the delta. A metric that:
- Stepped up/down sharply at `incident_start` → direct evidence
- Was already anomalous before `incident_start` → earlier upstream cause
- Is unchanged → not the cause, eliminate

Relevant metrics by incident class:
- Network: ALB `HTTPCode_Target_5XX_Count`, `TargetResponseTime`, TargetGroup `HealthyHostCount`
- Compute: EC2 `StatusCheckFailed`, ASG `GroupInServiceInstances`, Lambda `Throttles`/`Errors`
- Identity: CloudTrail `AccessDenied` rate, STS `AssumeRole` failures
- Data: RDS `DatabaseConnections`, `CPUUtilization`, `ReplicaLag`

### Axis 4 — Cross-account blast radius

**Unique to this dual-account setup**. Run the same alarm/metric check
on **the other China account** for the same time window. Three outcomes:

| Other account status | Interpretation |
|----------------------|---------------|
| **Also affected** | Likely shared upstream — AWS region event, shared service dependency, or identical misconfiguration deployed to both |
| **Not affected** | Account-scoped cause — credentials, account-specific deploy, account-specific resource |
| **Unknown (no comparable signal)** | Inconclusive — continue with account-scoped investigation |

This axis is the single biggest value-add vs. single-account RCA. Always
run it.

## Procedure

### Step 1 — Consume the Triage Card

Parse account, class, severity, resource, first-seen. If no card exists,
request one (or run triage first).

### Step 2 — Launch 4-axis parallel investigation

Fire all 4 axes concurrently. Do not serialize — Axis 2 deploy correlation
alone may involve 3–5 API calls.

### Step 3 — Synthesize

Rank the evidence by directness:
1. **Direct** — a CloudTrail event or deploy that modified the exact
   failing resource in the right time window
2. **Strong circumstantial** — metric step change at `incident_start`
   with no deploy correlation
3. **Weak** — metric drift over hours/days, multiple possible causes

Pick the **single most likely** root cause. If two candidates are roughly
equal, say so explicitly — don't pretend certainty.

### Step 4 — Produce the RCA Report

```
## Root Cause (hypothesis)
**<One sentence root cause>**
Confidence: <Direct / Strong / Weak>

## Evidence
1. <Axis 1 finding with timestamps and ARNs>
2. <Axis 2 finding>
3. <Axis 3 finding>
4. <Axis 4 finding — blast radius>

## Eliminated hypotheses
- <Thing you considered and ruled out, with reason>

## Blast radius
- <Account-scoped / Region-scoped / Partition-scoped>
- <Other account status>

## Recommended next step
→ Hand off to china-incident-mitigation with this RCA as input.
```

## Things not to do

- **Do not** execute remediation actions — this skill is read-only.
  Mitigation is a separate skill.
- **Do not** claim the cause without at least one piece of direct or
  strong evidence. "Probably X" with no evidence is a hypothesis, not a
  root cause.
- **Do not** skip Axis 4 (cross-account blast radius). It is cheap to
  check and eliminates the "is this widespread or just us?" question
  that every SRE asks next.
- **Do not** investigate beyond the 4 axes without stating why. Expanding
  scope during RCA dilutes the signal.
- **Do not** blame a correlated deploy without verifying it actually
  touched the failing resource. Every deploy is temporally correlated
  with something.
- **Do not** re-run triage inside RCA. Take the card as given.

## Examples

**Input**: Triage card = `aws-cn / Network / SEV-2 / prod-alb / 14:22`

**Action**: Parallel run the 4 axes.

Findings:
- Axis 1: CloudTrail shows `ModifyTargetGroupAttributes` at 14:19 by role
  `terraform-ci` — 3 min before alarm
- Axis 2: Terraform pipeline deployed stack `prod-alb` at 14:19 (matches)
- Axis 3: `HealthyHostCount` dropped from 4 to 0 at 14:20, recovered to 2
  at 14:25
- Axis 4: aws-cn-2 `prod-alb` unaffected

Hypothesis: "Terraform deploy at 14:19 changed target group
`deregistration_delay` from 30s to 300s, causing healthy hosts to be
marked draining and unavailable for ~5 min until connections drained."
Confidence: **Direct**.

Blast radius: aws-cn only. aws-cn-2 unaffected → not a region event.

**Input**: "为什么今天两个中国区账号都 Lambda 调用 AuthFailure"

**Action**: Investigation starts with the dual-account signal — that's
itself Axis 4 evidence. Skip direct to:
- Axis 1: CloudTrail `CreateAccessKey` / `UpdateAccessKey` in both accounts
- Axis 2: Did someone re-seed Secrets Manager in both within the same window?

Likely root cause: shared credential rotation event (e.g., operator
re-ran `aws secretsmanager update-secret` for both `/mcp/aws-cn` and
`/mcp/aws-cn-2` with a stale/mis-typed AK/SK).

Blast radius: Partition-scoped — both China accounts affected
simultaneously. Strong evidence of a shared-operator cause, not an AWS
infra event.
