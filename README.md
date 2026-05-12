# JIRA Test Tool Skills

Claude Code Skills for JIRA test management — defect creation and PRD-to-test-case conversion.

## Skills

### prd-to-jira

Generate structured test points from PRD documents, translate to English, and create JIRA/Xray test cases in one click.

- Trigger: "生成测试点", "PRD转JIRA", "创建测试用例", "写测试用例"
- Features: AI-powered test point generation, auto translation (ZH→EN), Xray integration, custom dictionary support

### defect-to-jira

Create JIRA defects (bugs) — single or batch from Excel, with automatic Chinese-to-English translation.

- Trigger: "创建缺陷", "提Bug", "报缺陷", "批量缺陷", "Excel缺陷"
- Features: Single/batch creation, Excel import, auto translation, template system, attachment support

## Installation

Copy skill directories to your Claude Code skills folder:

```bash
cp -r prd-to-jira ~/.claude/skills/
cp -r defect-to-jira ~/.claude/skills/
```

On first use, each skill will guide you through JIRA configuration (server URL, credentials, project key).

## File Structure

```
├── prd-to-jira/
│   ├── SKILL.md                          # Skill definition and workflow
│   └── references/
│       ├── config-setup.md               # Configuration guide
│       ├── jira-api-quickref.md          # JIRA/Xray API reference
│       └── output-format.md              # Output format specification
└── defect-to-jira/
    ├── SKILL.md                          # Skill definition and workflow
    └── references/
        ├── excel-format.md               # Excel parsing format spec
        ├── template-system.md            # Template management guide
        └── jira-defect-api.md            # JIRA defect API reference
```
