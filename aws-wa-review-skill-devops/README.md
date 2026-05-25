# aws-well-architected-review-skill-devops

Custom skill for **AWS DevOps Agent** that conducts an automated AWS Well-Architected Framework review across all six pillars and produces a structured Markdown report with prioritized remediation roadmap.

> ⚠️ **Distribution**: AWS DevOps Agent loads custom skills only as `.zip` uploads through the Operator Web App. The unzipped folder in this repo is for review/reading on GitHub. To use the skill, grab the matching `.zip` at the repo root (or zip this directory yourself with `SKILL.md` at the zip root) and upload it.

## What it does

- Runs **49 read-only AWS CLI checks** across all 6 Well-Architected pillars (Security, Reliability, Operational Excellence, Performance Efficiency, Cost Optimization, Sustainability)
- Classifies each finding with both:
  - **Severity** (🔴 CRITICAL / 🟠 HIGH / 🟡 MEDIUM / 🔵 LOW / ⚪ INFO)
  - **Fix Impact** (`downtime` / `slowness` / `additionalCost` / `needFullTest`, each `0`/`1`/`-1`)
- Maps Security findings to official WA BP IDs (`SECxx.BPxx`) for easy WA Tool sync
- Builds a 3-phase improvement roadmap (**0-30 days / 1-6 months / 6-24 months**) with paste-ready remediation CLI for every finding
- Optionally syncs results to AWS Well-Architected Tool via API (separate write credentials required)
- Bilingual: English by default, Chinese via [`references/SKILL_ZH.md`](references/SKILL_ZH.md)

## Trigger phrases

Activate this skill when the operator asks for:

- "Well-Architected review" / "WAFR" / "WA assessment"
- "Architecture review" / "AWS architecture audit"
- "Security assessment" / "security posture review"
- "Cost optimization audit" / "cost review"
- "Reliability check" / "Multi-AZ audit"
- "Performance audit"
- "Sustainability evaluation"
- Chinese: "架构评审" / "六大支柱评估" / "成本优化审计" / "可靠性检查" / "安全态势评估"

## Workflow at a glance

```
Phase 1: Bootstrap  (~2 min)     → Verify AWS CLI + credentials, mandatory read-only boundary check, scope confirmation
Phase 2: Assess     (~15-30 min) → 6 pillars in Security-First order, on-demand load each pillar's check file
Phase 3: Analyze    (~5 min)     → HRI/MRI/LRI classification, cross-pillar correlation, priority matrix
Phase 4: Report     (~2 min)     → Markdown report with 3-phase roadmap, Quick Wins, Top-10 implementation guide
```

Default mode is **autopilot** — only Phase 1 requires operator interaction.

## Required permissions

| Phase | IAM policy |
|-------|-----------|
| Assessment (read-only) | `arn:aws:iam::aws:policy/ReadOnlyAccess` **or** `ViewOnlyAccess` **or** `SecurityAudit` |
| Optional WA Tool sync (write) | `wellarchitected:*` actions — must be a **separate** credential, never mixed with the assessment credential |

The skill aborts if it detects `AdministratorAccess` / `PowerUserAccess` on the active credential — that's enforced by [`references/credential-boundary.md`](references/credential-boundary.md).

## Skill structure

```
aws-well-architected-review-skill-devops/
├── SKILL.md                          # Main entry (frontmatter + workflow)
├── README.md                         # This file
├── references/
│   ├── SKILL_ZH.md                   # Chinese mirror of SKILL.md
│   ├── workflow-overview.md          # Phase-by-phase detail
│   ├── environment-bootstrap.md      # Credential setup + DON'T-FETCH list
│   ├── credential-boundary.md        # Read-only boundary definition
│   ├── security-first-guide.md       # Why Security pillar runs first
│   ├── pillar-assessment-guide.md    # Top-5 rule + 4-subtheme grid + scoring rubric
│   ├── risk-classification.md        # Severity + Fix Impact + color contract
│   ├── mapping-table.md              # Pillar → Question → BP → local check mapping
│   ├── report-template.md            # Report layout
│   ├── wa-tool-sync.md               # AWS WA Tool API workflow (with 7 engineering pitfalls)
│   └── programmatic-checks/
│       ├── security-checks.md        # 12 Security checks
│       ├── reliability-checks.md     # 9 Reliability checks
│       ├── ops-excellence-checks.md  # 8 Ops Excellence checks
│       ├── performance-checks.md     # 7 Performance checks
│       ├── cost-checks.md            # 8 Cost Optimization checks
│       └── sustainability-checks.md  # 5 Sustainability checks
└── examples/
    └── sample-assessment.md          # Sample report output
```

**Total size**: ~150 KB unzipped (well under the 6 MB DevOps Agent upload limit).
**No `scripts/` directory** — DevOps Agent rejects skills that contain executable scripts. The agent generates the Markdown report directly from the templates.

## Recommended Agent Types

When uploading via the Operator Web App, target these Agent Types:

- ✅ **Generic** — broad applicability for architecture reviews
- ✅ **On-demand** — operators explicitly invoke "WA review"
- 🔸 **Evaluation** — useful when an evaluation pipeline measures architecture compliance
- ❌ **Incident Triage / RCA / Mitigation** — not designed for active incident response

## Source

Adapted from the standalone Claude Code / Kiro skill at:
[`aws-samples/sample-aws-resilience-skill/aws-well-architected-review`](https://github.com/aws-samples/sample-aws-resilience-skill/tree/master/aws-well-architected-review)

The DevOps Agent variant differs in the following ways:

- `SKILL.md` frontmatter uses only `name` + `description` (DevOps Agent ignores `allowed-tools` / `model`)
- No `scripts/` directory; HTML report generation removed (Markdown only)
- `description` rewritten in DevOps-Agent-style trigger language
- Single English entry point (`SKILL.md` at root), Chinese version moved to `references/SKILL_ZH.md` (matches the EKS Resilience Checker skill's pattern in this repo)

## How to upload

1. Zip this directory with `SKILL.md` at the zip root:
   ```bash
   cd Skills-ForDevOpsAgent/aws-wa-review-skill-devops
   zip -r ../aws-wa-review-skill-devops.zip .
   ```
2. Open the **AWS DevOps Agent Operator Web App** → Skills → **Add skill** → **Upload skill**
3. Drag-and-drop the `.zip`
4. Select Agent Types (`Generic` + `On-demand` recommended)
5. Click **Upload**

Full upload guide: [repo root README](../README.md).
