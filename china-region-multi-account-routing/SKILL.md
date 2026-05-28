---
name: china-region-multi-account-routing
description: Routing and disambiguation guidance for the two AWS China region
  MCP servers exposed by this Agent Space. Use this skill whenever the user's
  request mentions "中国区", "China", "cn-north-1", "cn-northwest-1", "Beijing",
  "Ningxia", or any AWS resource that must resolve to a specific China
  partition account. The skill explains which MCP endpoint maps to which
  account, how to pick when the user does not specify, and how to label
  cross-account results so the user can tell them apart.
---

# China Region Multi-Account Routing

Two separate MCP servers are registered for AWS China. Each backing pod runs
with its own AK/SK — credentials are **not** shared, so a request routed to
one cannot see resources in the other.

## Endpoint ↔ account ↔ region mapping

| MCP endpoint host         | Account (logical) | Region           |
|---------------------------|-------------------|------------------|
| `aws-cn.yingchu.cloud`    | aws-cn            | `cn-northwest-1` |
| `aws-cn-2.yingchu.cloud`  | aws-cn-2          | `cn-north-1`     |

Account IDs are intentionally not documented here. Users refer to accounts
by the logical names `aws-cn` / `aws-cn-2` or by region; the agent picks
the right MCP from those.

Both endpoints expose the same `awslabs.aws-api-mcp-server` tool surface.
The difference is **which account's credentials the backing pod holds**,
not the tool set.

## Routing rules

Apply in order; first match wins.

1. **User names an account explicitly** (e.g. "查 aws-cn-2 的 EC2",
   "宁夏那个账号", "the Beijing account") → route to that endpoint only.

2. **User names a region, not an account**:
   - "宁夏" / "cn-northwest-1" / "ningxia" → `aws-cn`
   - "北京" / "cn-north-1" / "beijing" → `aws-cn-2`

3. **User names neither** ("中国区 / China AWS / 中国的资源"): **ambiguous**.
   Do not silently pick. Choose one of:
   - **Preferred**: query both endpoints in parallel and return a merged
     response with each row labeled by account name.
   - **Alternative**: ask one disambiguation question before proceeding,
     e.g. "你指的是 aws-cn (宁夏) 还是 aws-cn-2 (北京)？还是两个都查？"

4. **Comparison / diff requests** ("对比两个账号", "哪个账号有 xxx",
   "两边 VPC 配置一样吗") → always query both, present side-by-side.

## Output format for cross-account results

Never merge results from the two accounts without attribution. Use one of:

**Side-by-side table** (preferred for ≤5 columns of data):

| Resource | aws-cn (Ningxia) | aws-cn-2 (Beijing) |
|----------|------------------|--------------------|
| VPC CIDR | 10.0.0.0/16      | 10.1.0.0/16        |

**Grouped sections** (preferred for lists):

```
### aws-cn (cn-northwest-1)
- i-0abc... (t3.medium, running)
- i-0def... (t3.small, stopped)

### aws-cn-2 (cn-north-1)
- i-0xyz... (t3.medium, running)
```

Every instance ID, ARN, or resource name you return must be prefixed or
suffixed with the account it came from — users have independent access to
each account, and an unattributed ARN is ambiguous (or worse, misleading).

## Things not to do

- **Do not** try to `sts:AssumeRole` from one China account into the other.
  The two MCP pods hold independent AK/SK; there is no trust relationship
  between these accounts.
- **Do not** assume aws-cn and aws-cn-2 share IAM principals, VPCs,
  Security Groups, or any named resources. Name collisions between the two
  accounts are coincidental.
- **Do not** silently fall back to aws-cn when aws-cn-2 returns an error
  (or vice versa). If the user asked about account X and X is unreachable,
  report the failure — do not answer with data from a different account.
- **Do not** cross-partition: these are both `aws-cn` partition accounts.
  Never use credentials from `aws-global` (us-*, eu-*, etc.) here; they
  will return `AuthFailure`.

## Examples

**Input**: "北京账号有几个 EC2 在跑"
**Action**: Route to `aws-cn-2` only. Run `ec2 describe-instances` with
filter `instance-state-name=running`. Report count and instance IDs.

**Input**: "中国区所有 S3 bucket 列一下"
**Action**: Ambiguous on account. Query both endpoints in parallel. Return:

| Bucket | Account | Region |
|---|---|---|
| my-logs-nw | aws-cn | cn-northwest-1 |
| my-logs-n | aws-cn-2 | cn-north-1 |

**Input**: "两个中国区账号 VPC CIDR 有没有冲突"
**Action**: Query both, extract CIDRs, compare, report overlap (if any) or
confirm no overlap. Always label which CIDR belongs to which account.
