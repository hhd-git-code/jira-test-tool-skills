# 缺陷模板系统说明

## 配置文件

位置：`~/.claude/defect-to-jira-config.json`

### 完整配置结构

```json
{
  "jiraDefect": {
    "serverUrl": "https://company.atlassian.net",
    "username": "user@company.com",
    "apiToken": "your-jira-api-token",
    "projectKey": "PROJ",
    "issueType": "Bug"
  },
  "customDict": [
    {"zh": "导航", "en": "Navigation"},
    {"zh": "蓝牙", "en": "Bluetooth"},
    {"zh": "车载信息娱乐系统", "en": "In-Vehicle Infotainment"},
    {"zh": "中控屏", "en": "Head Unit"},
    {"zh": "语音助手", "en": "Voice Assistant"},
    {"zh": "倒车影像", "en": "Rear View Camera"}
  ],
  "templates": [
    {
      "name": "IVI Audio Crash",
      "data": {
        "summary": "",
        "priority": "Blocker",
        "timestamp": "",
        "precondition": "Audio playing",
        "steps": "",
        "expectedResult": "",
        "actualResult": "",
        "reproduceRate": "Always",
        "recoverSteps": "Restart the Head Unit"
      }
    }
  ]
}
```

### 字段说明

#### jiraDefect

| 字段 | 必填 | 说明 |
|------|------|------|
| serverUrl | 是 | JIRA 服务器地址 |
| username | 是 | JIRA 账户邮箱 |
| apiToken | 是 | JIRA API Token |
| projectKey | 是 | 缺陷所在项目的 Key |
| issueType | 否 | 默认 "Bug" |

#### customDict

自定义中英词典，翻译时优先使用。数组格式，每项 `{"zh": "中文", "en": "English"}`。

翻译规则：
1. 先用 customDict 替换（最长匹配优先）
2. 再由 Claude 翻译剩余中文
3. Priority 和 ReproduceRate 值保持英文（中文自动映射）

#### templates

缺陷模板数组，每个模板包含：
- `name`: 模板名称（唯一标识）
- `data`: 9 个字段的预填值

模板 data 包含所有 9 个字段（summary, priority, timestamp, precondition, steps, expectedResult, actualResult, reproduceRate, recoverSteps）。空字符串 `""` 表示不预填该字段。

应用模板时，**只有非空字段**会作为默认值，用户输入可覆盖。

## 模板操作

### 列出模板

1. 读取配置文件
2. 按顺序列出模板名称，附带非空字段摘要
3. 示例输出：
   ```
   1. IVI Audio Crash — priority: Blocker, precondition: Audio playing, reproduceRate: Always, recoverSteps: Restart the Head Unit
   2. Bluetooth Connection — priority: Major, precondition: Bluetooth device available
   ```

### 新建模板

1. AskUserQuestion 询问模板名称
2. 确认名称不重复
3. AskUserQuestion 选择创建方式：
   - 从当前缺陷值创建（使用刚填写/刚创建的缺陷数据）
   - 从头填写
4. 从头填写时，分 3 轮收集 9 个字段的默认值：
   - 第 1 轮（4 个问题）：summary 默认值、priority 默认值、timestamp 默认值、precondition 默认值
   - 第 2 轮（4 个问题）：steps 默认值、expectedResult 默认值、actualResult 默认值、reproduceRate 默认值
   - 第 3 轮（1 个问题）：recoverSteps 默认值
5. 写入配置文件

### 编辑模板

1. 列出所有模板名称
2. AskUserQuestion 选择要编辑的模板
3. AskUserQuestion 询问要修改哪些字段（多选）
4. 对每个选中的字段，AskUserQuestion 输入新值
5. 更新配置文件

### 删除模板

1. 列出所有模板名称
2. AskUserQuestion 选择要删除的模板
3. 确认后从 templates 数组移除
4. 更新配置文件

## 首次配置流程

1. 确认 `~/.claude/defect-to-jira-config.json` 不存在或缺少必填字段
2. 通过 AskUserQuestion 收集全部配置：
   - 第 1 轮：JIRA server URL、JIRA username
   - 第 2 轮：JIRA API Token、JIRA project key
   - 第 3 轮：Issue type（默认 Bug）、是否配置自定义词典
3. 执行 JIRA 连接测试验证凭证
4. 验证通过后用 Write 工具写入配置文件
5. customDict 和 templates 初始为空数组
6. 验证失败不写入，提示用户修正后重试

## 安全提示

- 配置文件包含 API Token，不要提交到 Git
- API Token 通过环境变量传递给 curl，避免暴露在命令行日志中
