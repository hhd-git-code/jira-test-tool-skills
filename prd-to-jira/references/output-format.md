# 测试点输出格式

## test-points.zh.md（中文）

```markdown
# [PRD 名称] 测试点

## TC-001: [标题]

- **优先级**: Major
- **描述**: [测试目的和验证内容]
- **前置条件**:
  1. [条件1]
  2. [条件2]
- **测试步骤**:
  | 步骤 | 操作 | 预期结果 |
  |------|------|----------|
  | 1 | [具体操作] | [预期结果] |
  | 2 | [具体操作] | [预期结果] |

---

## TC-002: [标题]

- **优先级**: Critical
- **描述**: [测试目的和验证内容]
- **前置条件**:
  1. [条件1]
- **测试步骤**:
  | 步骤 | 操作 | 预期结果 |
  |------|------|----------|
  | 1 | [具体操作] | [预期结果] |

---
```

规则：
- 编号从 TC-001 开始，顺序递增
- 优先级取值：Blocker、Critical、Major、Minor、Trivial
- 每个测试点之间用 `---` 分隔
- 前置条件用有序列表
- 测试步骤用 Markdown 表格
- 所有内容使用中文

## test-points.en.md（双语）

```markdown
# [PRD 名称] Test Points (Bilingual)

## TC-001: [中文标题]

- **Priority**: Major
- **Title (EN)**: [English title]
- **Description (EN)**: [English description]
- **Precondition (EN)**:
  1. [English precondition 1]
  2. [English precondition 2]
- **Test Steps**:
  | Step | Action (EN) | Expected Result (EN) |
  |------|------------|---------------------|
  | 1 | [action in English] | [expected result in English] |
  | 2 | [action in English] | [expected result in English] |

---

## TC-002: [中文标题]

- **Priority**: Critical
- **Title (EN)**: [English title]
- **Description (EN)**: [English description]
- **Precondition (EN)**:
  1. [English precondition 1]
- **Test Steps**:
  | Step | Action (EN) | Expected Result (EN) |
  |------|------------|---------------------|
  | 1 | [action in English] | [expected result in English] |

---
```

规则：
- 保留中文标题作为 section header（便于对照）
- 英文字段后缀 `(EN)` 标识
- Priority 保持英文（与 JIRA 优先级名称一致）
- 所有需要翻译的字段都提供英文版本

## 覆盖对照表

文件末尾附加覆盖对照表，方便确认 PRD 功能点是否都有对应测试点：

```markdown
## 覆盖对照表

| 功能点 | 测试点编号 |
|--------|-----------|
| [功能点1] | TC-001, TC-002 |
| [功能点2] | TC-003, TC-004, TC-005 |
```
