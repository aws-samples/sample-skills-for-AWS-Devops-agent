---
name: china-incident-mitigation
description: Draft step-by-step mitigation CLI commands for a root-caused
  incident in either China account (aws-cn or aws-cn-2). Use this skill
  after RCA has identified the root cause, when the user asks for
  mitigation, remediation, 缓解, 修复, 回滚, rollback, restore service,
  怎么修, fix it, 怎么办. Covers common mitigation patterns such as
  credential rotation, Kubernetes pod rollout-restart, ALB target group
  reattach, security group rule revoke, IAM policy rollback, and safe
  CloudFormation stack rollback. Output always includes the exact CLI
  command, a one-line explanation of what it changes, a rollback/undo
  command, and an explicit human approval prompt. CRITICAL — this skill
  NEVER executes commands autonomously; every mitigation step requires
  explicit user approval before running.
---

# China Incident Mitigation

Routing is governed by `china-region-multi-account-routing`.
RCA hand-off is governed by `china-incident-rca`. This skill produces
**command drafts for human approval**, not autonomous actions.

## Intended agent type

Upload with Agent Type **Incident Mitigation** selected.

## The approval contract

This skill **only produces command drafts for human review**. It never
executes write operations autonomously.

Enterprise deployments: route approvals through your change management
system (ServiceNow, Jira, OpsGenie, etc.). The agent outputs the command;
a human approves and executes it in the change window.

Operator/demo deployments: the agent will wait for explicit "approve step N"
before executing. But even then, treat this as a convenience shortcut —
never in production without a change record.

Every mitigation output in this skill uses this exact 4-field format.
No exceptions.

```
### Mitigation step N — <short name>

Command:
  aws <command> --region <region> --profile <account>  \
    <args>

What it does:
  <one sentence describing the state change>

Rollback:
  aws <undo command> --region <region> --profile <account>  \
    <args>

Approval required:
  This command changes production state in <account>.
  Enterprise: submit to change management system before executing.
  Operator: reply "approve step N" to execute, "skip N" to skip,
  or "stop" to abort.
```

The agent **never** executes without explicit approval.
"yes", "do it", "go ahead" are NOT valid — must be "approve step N"
or a documented change record approval.

## Pattern library

Map RCA findings to known mitigation patterns. If the RCA does not match
any pattern, fall back to general investigation (do not improvise
mitigation).

### Pattern A — Credential failure (AuthFailure / ExpiredToken)

Root cause signal: MCP pod logs show `AuthFailure` or `ExpiredToken`.

Steps:
1. Verify credentials in Secrets Manager are current
   ```
   aws secretsmanager get-secret-value \
     --secret-id /mcp/aws-cn --region us-east-1
   ```
2. If stale, rotate via operator — **user provides the new AK/SK**, agent
   does not generate credentials
   ```
   aws secretsmanager put-secret-value \
     --secret-id /mcp/aws-cn --region us-east-1 \
     --secret-string '{"AK":"<NEW>","SK":"<NEW>"}'
   ```
3. Force pod restart to pick up new secret
   ```
   kubectl -n mcp rollout restart deploy/aws-cn
   ```

### Pattern B — MCP pod crashloop

Root cause signal: pod RESTARTS > 0, recent image change.

Steps:
1. Inspect recent crash
   ```
   kubectl -n mcp logs deploy/aws-cn --previous
   ```
2. If image rollout is the cause, roll back
   ```
   kubectl -n mcp rollout undo deploy/aws-cn
   ```
3. Confirm healthy
   ```
   kubectl -n mcp rollout status deploy/aws-cn --timeout=2m
   ```

### Pattern C — ALB target unhealthy

Root cause signal: ALB 5xx spike, target group health check failing.

Steps:
1. Describe current target health
   ```
   aws elbv2 describe-target-health \
     --target-group-arn <tg-arn> --region cn-northwest-1
   ```
2. If pods are ready but ALB marks them unhealthy, check recent
   `ModifyTargetGroupAttributes` from RCA evidence and revert:
   ```
   aws elbv2 modify-target-group-attributes \
     --target-group-arn <tg-arn> --region cn-northwest-1 \
     --attributes Key=<attr>,Value=<previous-value>
   ```
3. Rollback command (if revert itself breaks):
   ```
   aws elbv2 modify-target-group-attributes \
     --target-group-arn <tg-arn> --region cn-northwest-1 \
     --attributes Key=<attr>,Value=<current-value>
   ```

### Pattern D — Overly-permissive SG (security incident)

Root cause signal: SG rule `0.0.0.0/0` on sensitive port added
accidentally.

Steps:
1. Revoke the rule
   ```
   aws ec2 revoke-security-group-ingress \
     --group-id <sg-id> --region <region> \
     --protocol tcp --port <port> --cidr 0.0.0.0/0
   ```
2. Rollback command (if this rule was intentional and you just revoked a
   production-needed rule):
   ```
   aws ec2 authorize-security-group-ingress \
     --group-id <sg-id> --region <region> \
     --protocol tcp --port <port> --cidr 0.0.0.0/0
   ```
3. Verify
   ```
   aws ec2 describe-security-groups \
     --group-ids <sg-id> --region <region>
   ```

### Pattern E — CloudFormation stack stuck

Root cause signal: CFN stack in `UPDATE_ROLLBACK_FAILED` or
`UPDATE_IN_PROGRESS` too long.

Steps:
1. Inspect
   ```
   aws cloudformation describe-stack-events \
     --stack-name <stack> --region <region> --max-items 20
   ```
2. Continue rollback skipping failed resources (surgical):
   ```
   aws cloudformation continue-update-rollback \
     --stack-name <stack> --region <region> \
     --resources-to-skip <logical-id>
   ```
3. Rollback option: none — CFN rollback is itself the rollback. Do not
   delete-stack unless the user explicitly authorizes (data loss risk).

### Pattern F — Secrets Manager secret accidentally deleted

Root cause signal: `ResourceNotFoundException` when MCP pod reads secret.

Steps:
1. Check if recoverable (7-day window)
   ```
   aws secretsmanager restore-secret \
     --secret-id /mcp/aws-cn --region us-east-1
   ```
2. If past recovery window, recreate and repopulate (user provides
   AK/SK — agent does not generate)
   ```
   aws secretsmanager create-secret \
     --name /mcp/aws-cn --region us-east-1 \
     --secret-string '{"AK":"<NEW>","SK":"<NEW>"}'
   ```

## Procedure

1. Read the RCA output
2. Match to a pattern in the library
3. Produce each mitigation step in the 4-field format
4. Wait for explicit "approve step N" before executing
5. After each step, report result and wait for next approval

## Things not to do

- **Do not** execute any command without the exact phrase "approve step N".
  Ambiguous responses ("sure", "yes", "ok") → ask again, do not execute.
- **Do not** generate AWS credentials (AK/SK). The agent cannot see
  customer root-level credential creation. User must provide.
- **Do not** bundle multiple steps into a single approval. Each step is
  approved individually so the user can abort at any point.
- **Do not** improvise a mitigation if the RCA does not match a library
  pattern. Instead, state: "This pattern is not in the mitigation library.
  Recommend manual operator action." Then offer to help draft a one-off
  command with the user's confirmation of each field.
- **Do not** run `aws cloudformation delete-stack` as a mitigation —
  too often used to "fix" stuck stacks when `continue-update-rollback`
  is safer.
- **Do not** use `kubectl delete pod` when `kubectl rollout restart deploy`
  does the same thing with deployment-managed replacement.
- **Do not** silently retry failed commands. If step N fails, report the
  failure verbatim and wait for user decision.

## Examples

**Input**: RCA hypothesis = "Pattern A — Credential failure in aws-cn"

**Output**:
```
### Mitigation step 1 — Verify current credentials

Command:
  aws secretsmanager get-secret-value \
    --secret-id /mcp/aws-cn --region us-east-1

What it does:
  Reads the current AK/SK from Secrets Manager. Read-only.

Rollback:
  None needed (read-only).

Approval:
  This command reads production credentials in aws-cn. Reply
  "approve step 1" to execute, "skip 1" to move on, or "stop" to abort.
```

After approval + execution, the agent reports result and emits step 2.

**Input**: RCA hypothesis = "unknown — signal doesn't fit any pattern"

**Output**:
```
The RCA finding does not match any pattern in the mitigation library
(Patterns A–F). I will not auto-draft a mitigation. Options:

1. Describe the failure in more detail — I may be able to match a
   pattern I missed.
2. Draft a one-off mitigation together, field-by-field, with explicit
   confirmation at each step.
3. Escalate to a human operator.

Which would you like?
```
