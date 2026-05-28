# Skills for AWS DevOps Agent

> ## ⚠️ **IMPORTANT — READ FIRST**
>
> **AWS DevOps Agent custom skills MUST be uploaded as a `.zip` file** through the AWS DevOps Agent Operator Web App. The agent does **not** load skills from a Git repository or an unzipped directory.
>
> The unzipped skill folders checked into this repo (e.g. `eks-resilience-checker-skill-devops/`) are provided **for reference and review only** — so you can read the `SKILL.md`, references, and assets directly on GitHub. To actually use a skill, download/grab the corresponding **`.zip`** at the repo root (or zip the directory yourself, with `SKILL.md` at the zip root) and upload it via the Operator Web App.

A collection of custom skills built specifically for [**AWS DevOps Agent**](https://docs.aws.amazon.com/devopsagent/latest/userguide/about-aws-devops-agent.html). These are **not** generic Agent Skills — they are tailored to the AWS DevOps Agent runtime, its agent types, and the operational workflows it executes inside an Agent Space.

## Skills in this repo

| Skill | Description |
|-------|-------------|
| `eks-resilience-checker-skill-devops` | Investigation and remediation procedures for assessing the resiliency posture of Amazon EKS workloads (control plane, data plane, networking, autoscaling, observability, fault-injection mapping). |
| `aws-wa-review-skill-devops` | Automated AWS Well-Architected Framework review across all 6 pillars. Runs 55 read-only AWS CLI checks (Security, Reliability, Ops Excellence, Performance, Cost, Sustainability), classifies findings by severity + fix-impact, maps Security findings to official WA BP IDs, and produces a Markdown report with a 0-30d / 1-6m / 6-24m roadmap and paste-ready remediation CLI for every finding. |
| `china-region-multi-account-routing` | Routing layer: determines which MCP endpoint (aws-cn / aws-cn-2) to use based on region and account context. Triggers on China region keywords (cn-north-1, cn-northwest-1, 宁夏, 北京). |
| `china-incident-triage` | Alert first-response: classifies severity, deduplicates, and routes China-region incidents. |
| `china-incident-rca` | Root cause analysis: correlates CloudTrail events, deployment timelines, and metric anomalies to identify incident causes. |
| `china-incident-mitigation` | Mitigation recommendations: outputs CLI commands with mandatory human approval before execution. Never auto-executes write operations. |
| `china-account-prevention-checks` | Preventive inspections: single points of failure, quota limits, certificate expiry, credential aging. |
| `cn-partition-arn-routing` | ARN partition mismatch diagnostics (`arn:aws:` vs `arn:aws-cn:`) — catches common AccessDenied errors caused by wrong partition. |
| `cross-account-cost-attribution` | Dual-account cost comparison and attribution across China-region accounts. |
| `cross-account-inventory-compare` | Cross-account resource inventory comparison to detect drift and inconsistencies. |
| `cross-account-security-posture-check` | Security compliance audit: public S3 buckets, MFA status, open security group ports, and other risk indicators across accounts. |
| `use-eks-via-call-kubectl` | Guides the agent to use `call_kubectl` (not `use_kubectl`) when querying EKS clusters. |

Each skill is distributed as a `.zip` ready to upload via the AWS DevOps Agent Operator Web App.

## How these skills differ from generic Agent Skills

AWS DevOps Agent supports a **subset** of the open [Agent Skills specification](https://agentskills.io/). The skills in this repository are designed around those constraints, so they do not necessarily work as-is in a generic agent runtime:

- **Non-executable content only.** AWS DevOps Agent only loads Markdown instructions, PDFs, images, and data files. Skills containing a `scripts/` directory are **rejected** at upload time. You will not find executable scripts here — investigation steps are expressed as Markdown procedures the agent follows.
- **Mandatory `SKILL.md` with frontmatter.** Every skill ships a top-level `SKILL.md` whose YAML frontmatter (`name`, `description`) is what the agent uses to decide *when* to activate the skill. Descriptions are written from the agent's perspective and enumerate the symptoms / services / error types that should trigger the skill.
- **Agent-type targeting.** Skills here are intended to be uploaded with specific Agent Types selected (`Generic`, `On-demand`, `Incident Triage`, `Incident RCA`, `Incident Mitigation`, or `Evaluation`) so the agent only loads them in the right context, reducing context consumption and improving focus.
- **Operational workflow shape.** Content is organized as decision trees and step-by-step investigation playbooks (check alarm status → analyze metrics → identify root cause → summarize findings) rather than free-form prose, so the agent can follow them deterministically during incident response.
- **Designed to amplify built-in + MCP tools.** Skills document *when* the agent should call CloudWatch / EKS / MCP-server tools, *what* parameters to use, and *how* to interpret results — they are not standalone runnable programs.
- **6 MB upload limit.** Each `.zip` (including all references and assets) stays under the 6 MB hard limit enforced by the AWS DevOps Agent upload validator.

## Skill structure

```
<skill-name>.zip
├── SKILL.md              # Required: main instructions + YAML frontmatter
├── references/           # Optional: reference documentation
│   ├── architecture.md
│   └── troubleshooting.md
└── assets/               # Optional: images, diagrams, data files
```

`SKILL.md` frontmatter example (required for zip-uploaded skills):

```markdown
---
name: eks-resilience-checker-skill-devops
description: Investigation procedures for Amazon EKS resiliency assessment
  including control plane health, data plane scaling, networking, and
  observability gaps. Use this skill when assessing or remediating EKS
  cluster resiliency posture.
---
```

## How to upload a skill to AWS DevOps Agent

> Source: [AWS DevOps Agent — DevOps Agent Skills (official docs)](https://docs.aws.amazon.com/devopsagent/latest/userguide/about-aws-devops-agent-devops-agent-skills.html)

**Prerequisite:** you must already have an Agent Space in AWS DevOps Agent ([Creating an Agent Space](https://docs.aws.amazon.com/devopsagent/latest/userguide/getting-started-with-aws-devops-agent-creating-an-agent-space.html)).

1. Download the desired `*.zip` from this repository (or build your own following the structure above).
2. Make sure `SKILL.md` is at the **root** of the zip and includes valid frontmatter (`name`, `description`).
3. Confirm the zip contains **no `scripts/` directory** and is **≤ 6 MB** total.
4. Open the **AWS DevOps Agent Operator Web App** and navigate to the **Skills** page.
5. Click **Add skill**.
6. In the modal, select **Upload skill**.
7. Drag-and-drop the `.zip` file (ZIP only, max 6 MB) or click to browse.
8. Select one or more **Agent Types** that should use this skill. `Generic` is selected by default and applies to all agent types; deselect it to target specific agents (`On-demand`, `Incident Triage`, `Incident RCA`, `Incident Mitigation`, or `Evaluation`).
9. Review the validation results displayed by the upload form.
10. Click **Upload** to add the skill to your Agent Space.

### Managing uploaded skills

- **List / view** — go to the Skills page, click any skill to see its file tree (`SKILL.md`, `references/`, `assets/`) in read-only mode.
- **Activate / deactivate** — open the skill detail view and toggle `Active` ↔ `Inactive`. Inactive skills are preserved but not loaded for new investigations.
- **Re-target agents** — edit the skill's Agent Type selection at any time.
- **Replace** — for zip-uploaded skills, upload a new version to update the contents.

### Alternative: create a skill directly in the UI

If you only need a single `SKILL.md` (no extra references or assets), you can skip the zip flow:

1. Skills page → **Add skill** → **Create skill**.
2. Fill in **Name** (lowercase letters, numbers, hyphens; ≤ 64 chars; cannot start/end with hyphen).
3. Fill in **Description** (recommend ≥ 100 chars, ≤ 1024 chars).
4. Choose **Status** (`Active` / `Inactive`) and one or more **Agent Types**.
5. Paste your Markdown into **Instructions**.
6. Click **Create**. The system auto-generates `SKILL.md` with proper frontmatter.

## References

- [About AWS DevOps Agent](https://docs.aws.amazon.com/devopsagent/latest/userguide/about-aws-devops-agent.html)
- [DevOps Agent Skills](https://docs.aws.amazon.com/devopsagent/latest/userguide/about-aws-devops-agent-devops-agent-skills.html)
- [Learned Skills](https://docs.aws.amazon.com/devopsagent/latest/userguide/about-aws-devops-agent-learned-skills.html)
- [Agent Skills specification](https://agentskills.io/)

## License

This project is licensed under the MIT License — see the [LICENSE](LICENSE) file for details.
