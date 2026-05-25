# Jira Test Tool Skills

两个 Claude Code 技能，用于将测试工作流自动化到 JIRA。

## Skills

### defect-to-jira

创建 JIRA 缺陷（Bug），支持单条创建和 Excel 批量导入，自动将中文内容翻译为英文。

- 单条缺陷创建
- Excel 批量导入
- 中英文自动翻译
- JIRA 连接验证

### prd-to-jira

根据 PRD 文档生成结构化测试点，翻译为英文，一键创建 JIRA/Xray 测试用例。

- PRD 解析生成测试点
- 测试用例结构化输出
- JIRA/Xray 自动创建
- 中英文自动翻译

## 安装

将 `defect-to-jira/` 和 `prd-to-jira/` 目录复制到 `~/.claude/skills/` 下即可。
