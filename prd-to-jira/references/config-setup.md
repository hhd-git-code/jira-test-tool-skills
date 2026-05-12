# prd-to-jira 配置说明

## 配置文件位置

`~/.claude/prd-to-jira-config.json`

## 配置结构

```json
{
  "jiraTestCase": {
    "serverUrl": "https://company.atlassian.net",
    "username": "user@company.com",
    "apiToken": "your-jira-api-token",
    "projectKey": "PROJ",
    "issueType": "Test"
  },
  "xray": {
    "enabled": true,
    "clientId": "your-xray-client-id",
    "clientSecret": "your-xray-client-secret"
  },
  "customDict": [
    {"zh": "导航", "en": "Navigation"},
    {"zh": "蓝牙", "en": "Bluetooth"},
    {"zh": "车载信息娱乐系统", "en": "In-Vehicle Infotainment"},
    {"zh": "中控屏", "en": "Head Unit"},
    {"zh": "语音助手", "en": "Voice Assistant"},
    {"zh": "倒车影像", "en": "Rear View Camera"}
  ]
}
```

## 字段说明

### jiraTestCase

| 字段 | 必填 | 说明 |
|------|------|------|
| serverUrl | 是 | JIRA 服务器地址，如 `https://company.atlassian.net` |
| username | 是 | JIRA 账户邮箱 |
| apiToken | 是 | JIRA API Token（从 https://id.atlassian.com/manage-profile/security/api-tokens 创建） |
| projectKey | 是 | 测试用例所在项目的 Key，如 `PROJ` |
| issueType | 否 | 测试用例的 Issue Type，默认 `"Test"` |

### xray

| 字段 | 必填 | 说明 |
|------|------|------|
| enabled | 是 | 是否启用 Xray 集成 |
| clientId | 启用时必填 | Xray Cloud Client ID |
| clientSecret | 启用时必填 | Xray Cloud Client Secret |

Xray 凭证从 Xray Cloud Settings → API Keys 获取。

### customDict

自定义中英词典，翻译时优先使用词典中的翻译。数组格式，每项 `{"zh": "中文", "en": "English"}`。

## 首次设置流程

1. 检测配置文件是否存在
2. 不存在时，通过 AskUserQuestion 逐项收集：
   - JIRA server URL
   - JIRA username
   - JIRA API Token
   - JIRA project key
   - Issue type（默认 Test）
   - 是否使用 Xray？如果使用，收集 clientId 和 clientSecret
3. 执行 JIRA 连接测试验证凭证
4. 如果启用 Xray，执行 Xray 认证测试
5. 写入配置文件

## 更新配置

用户说"更新配置"、"重新配置"、"修改 JIRA 配置"时，重新运行首次设置流程，覆盖现有配置文件。

## 安全提示

- 配置文件包含 API Token 和 Xray Secret，不要提交到 Git
- 建议在 `.gitignore` 中添加 `prd-to-jira-config.json`
