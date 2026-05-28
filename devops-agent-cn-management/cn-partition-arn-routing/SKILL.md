---
name: cn-partition-arn-routing
description: Diagnose and explain AWS partition ARN mismatches in China region
  accounts. Use this skill whenever an investigation in `cn-north-1` or
  `cn-northwest-1` involves ARNs starting with `arn:aws:` instead of
  `arn:aws-cn:`, or whenever symptoms include AccessDenied, AuthFailure,
  MalformedPolicyDocument, NoSuchEntity, or "principal cannot be assumed"
  errors against IAM roles, SNS topics, KMS keys, S3 buckets, or any other
  ARN-bearing resource. Triggers also include the user mentioning "partition",
  "aws-cn vs aws", "cross-partition", "中国区 ARN 不对", "global partition
  ARN in China account", "trust policy 写错了", or pasting any ARN that
  looks like `arn:aws:iam::*` while the account context is China.
  Importantly, use this skill BEFORE concluding that an IAM trust policy
  or resource policy is "missing permissions" — the more common root cause in
  China region accounts is a partition string mismatch that the agent and
  generic LLM debuggers consistently get wrong.
---

# Cross-Partition ARN Diagnosis for AWS China Regions

## TL;DR for the agent

When you see an `AccessDenied` / `AuthFailure` / `cannot assume role` error
in a China region account, the **most common root cause is not "missing
permissions" — it is a partition mismatch in the ARN**. Check this before
suggesting any policy edits.

The error symptom looks like a permissions problem. The cause is a name
that doesn't resolve. They are not the same fix. Recommending the wrong
one wastes the user's time and may make the configuration worse.

## What partitions are

AWS is split into isolated **partitions**. ARNs encode the partition in
their second segment:

| Partition | Where | ARN prefix |
|-----------|-------|------------|
| Global    | All commercial regions outside China | `arn:aws:` |
| China     | `cn-north-1` (Beijing), `cn-northwest-1` (Ningxia) | `arn:aws-cn:` |
| GovCloud  | `us-gov-east-1`, `us-gov-west-1` | `arn:aws-us-gov:` |

China region resources **only** accept `arn:aws-cn:` ARNs. A trust policy,
resource policy, IAM policy, or SNS topic policy in a China account that
references `arn:aws:` is referencing a resource in the global partition,
which the China partition cannot see — so the principal/resource appears
to "not exist" even when an identically-named one exists in the same China
account.

## Detection rules

Apply these checks before concluding anything else.

### Rule 1 — account context is China, ARN is global

If the investigation involves resources in `cn-north-1` or `cn-northwest-1`
(or accounts whose own ARNs start with `arn:aws-cn:`), and you see any
embedded ARN that starts with `arn:aws:` (not `arn:aws-cn:`):
**this is the bug**, regardless of any other plausible-looking issue.

Examples that look like this:

```
"Principal": { "AWS": "arn:aws:iam::123456789012:role/some-role" }
                       ^^^^^^^^^^ — wrong partition, account is China
```

```
"Resource": "arn:aws:s3:::my-bucket-cn"
             ^^^^^^^^^^ — wrong partition for a China-region bucket
```

### Rule 2 — symptom-to-cause mapping

| Symptom                                            | Plausible-but-wrong cause     | Likely correct cause |
|----------------------------------------------------|-------------------------------|----------------------|
| `AccessDenied: User X is not authorized`           | "Add `s:Action` to policy"    | Wrong-partition principal in trust policy |
| `MalformedPolicyDocument: Invalid principal`       | "Principal field formatting"  | Wrong-partition ARN |
| `Cannot assume role`                               | "Add `sts:AssumeRole` to caller" | Wrong-partition role ARN |
| `NoSuchEntity` on a role you can see in the console | "Eventual consistency"        | Caller is using `aws:` ARN but role lives in `aws-cn` |
| KMS `AccessDenied` in cn region                    | "Key policy missing grant"    | Key policy references global partition principal |

### Rule 3 — never trust string match alone, check both ends

If the investigation pulls an IAM role's trust policy, **also** pull the
role's own ARN and confirm both are `aws-cn`. A role whose ARN is
`arn:aws-cn:iam::*:role/foo` cannot be assumed by a principal expressed
as `arn:aws:iam::*:user/bar` — even if `user/bar` exists in the same
account under `arn:aws-cn:iam::*:user/bar`, it is a different identity
from the partition's perspective.

## What to output when you find this

Structure the RCA exactly as follows. The user has been bitten by agents
suggesting the wrong fix, so be explicit.

```
Root cause: cross-partition ARN mismatch.

Account context: <region>, partition aws-cn
Offending ARN:   <the bad ARN>
Where it appears: <trust policy of role X / resource policy of bucket Y / etc.>
Why this fails:  ARNs in China region accounts must use the aws-cn
                 partition prefix. The reference to aws: is treated as
                 a global-partition resource that does not exist from
                 this account's perspective. Adding permissions does
                 not fix this — the principal is unreachable, not
                 unauthorized.

Fix: change the ARN partition segment from `aws` to `aws-cn`.

  Before:  arn:aws:iam::*:role/some-role
  After:   arn:aws-cn:iam::*:role/some-role
```

Then offer the corrected policy as a diff or full document, and ask
the user to confirm before applying.

## Anti-patterns to avoid

The wrong responses an agent often produces here:

1. **"Add `sts:AssumeRole` to the trust policy"** — the trust policy
   may already be correct in shape; the principal ARN is what's wrong.

2. **"Try detaching and reattaching the policy"** — flapping the
   attachment doesn't change the ARN string.

3. **"This is an IAM eventual-consistency issue, retry"** — if the
   ARN partition is wrong, retrying never resolves it.

4. **"Use a wildcard principal like `*` to test"** — masks the bug
   and creates a security hole.

5. **"Reference the role by name only (`role/foo`)"** — IAM trust
   policies require fully-qualified ARNs in most fields.

## When this skill does not apply

- If the account is in a global region (e.g. `us-east-1`) and the ARN
  is `arn:aws:`, that's correct. Look elsewhere.
- If both ARNs are already `arn:aws-cn:` and the error persists, the
  problem is genuinely a permissions or attachment issue — fall through
  to your normal IAM RCA approach.
- If the error is `ExpiredToken` / `InvalidClientTokenId`, that's a
  credentials issue (see `china-region-multi-account-routing` skill),
  not a partition issue.

## Tested cases

- Lambda assumes role with `arn:aws:` trust policy in `cn-north-1` →
  fails with `AccessDenied`. Changing to `arn:aws-cn:` resolves.
- Cross-account S3 bucket policy in `cn-northwest-1` references
  `arn:aws:` principal → bucket reads fail with `AccessDenied`. Changing
  to `arn:aws-cn:` resolves.
- KMS key policy in cn account uses `arn:aws:` for principal → encrypt
  calls fail. Changing partition resolves.
