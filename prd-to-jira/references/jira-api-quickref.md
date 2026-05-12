# JIRA/Xray API 速查

所有命令使用 curl，凭证从 `~/.claude/prd-to-jira-config.json` 读取。

## 通用变量

```bash
SERVER_URL="https://company.atlassian.net"
USERNAME="user@company.com"
API_TOKEN="your-api-token"
PROJECT_KEY="PROJ"
```

## 1. 连接测试

```bash
curl -s -w "\n%{http_code}" \
  -u "$USERNAME:$API_TOKEN" \
  "$SERVER_URL/rest/api/2/myself"
```

成功返回 200。

## 2. 获取优先级列表

```bash
curl -s \
  -u "$USERNAME:$API_TOKEN" \
  "$SERVER_URL/rest/api/2/priority"
```

返回 JSON 数组，每个元素有 `id` 和 `name`。用于将优先级名称（如 "Major"）映射为 ID。

## 3. 标准路径：创建 JIRA Issue

```bash
curl -s -X POST \
  -u "$USERNAME:$API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "fields": {
      "project": {"key": "'"$PROJECT_KEY"'"},
      "summary": "<titleEn>",
      "description": "<wikiMarkupDescription>",
      "issuetype": {"name": "Test"},
      "priority": {"name": "Major"}
    }
  }' \
  "$SERVER_URL/rest/api/2/issue"
```

成功返回：`{"id":"12345","key":"PROJ-123"}`

### JIRA Description Wiki Markup 格式

```
h2. Description
{descriptionEn}

h2. Preconditions
{preconditionEn}

h2. Test Steps
||Step||Action||Expected Result||
|1|{actionEn}|{expectedResultEn}|
|2|{actionEn}|{expectedResultEn}|
```

注意：
- wiki markup 中换行用 `\n`
- 表格行以 `|` 分隔
- 表头行以 `||` 分隔
- 特殊字符需要转义：`{` → `\{`, `}` → `\}`, `|` → 在表格内不能转义，需避免使用
- 优先级如果用 ID 而非名称：`"priority": {"id": "3"}`

## 4. Xray 路径

### 4.1 认证获取 Bearer Token

```bash
curl -s -X POST \
  -H "Content-Type: application/json" \
  -d '{"client_id":"'"$XRAY_CLIENT_ID"'","client_secret":"'"$XRAY_CLIENT_SECRET"'"}' \
  https://xray.cloud.getxray.app/api/v1/authenticate
```

返回纯字符串 token（带引号），如 `"eyJhbGciOiJIUzI1NiJ9..."`。去掉外层引号即为 Bearer token。

### 4.2 获取项目信息

```bash
curl -s \
  -u "$USERNAME:$API_TOKEN" \
  "$SERVER_URL/rest/api/2/project/$PROJECT_KEY"
```

从响应中提取：
- `id` → projectId
- `issueTypes[]` → 找 `name == "Test"` 的 id 作为 testIssueTypeId
- `issueTypes[]` → 找 `name == "Precondition"` 的 id 作为 preconditionIssueTypeId

如果找不到 "Test" issue type，说明 Xray 未在该项目启用。

### 4.3 创建 Test（含手动步骤）

```bash
XRAY_TOKEN="eyJhbGciOiJIUzI1NiJ9..."
PROJECT_ID="10000"
TEST_ISSUE_TYPE_ID="10100"

curl -s -X POST \
  -H "Authorization: Bearer $XRAY_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "mutation createTest($jira: JSON!, $steps: [CreateStepInput]) { createTest(testType: { name: \"Manual\" }, steps: $steps, jira: $jira) { test { issueId jira(fields: [\"key\"]) } warnings } }",
    "variables": {
      "jira": {
        "fields": {
          "summary": "<titleEn>",
          "project": {"id": "'"$PROJECT_ID"'"},
          "issuetype": {"id": "'"$TEST_ISSUE_TYPE_ID"'"},
          "description": "<descriptionEn>"
        }
      },
      "steps": [
        {"action": "<actionEn step1>", "result": "<expectedResultEn step1>"},
        {"action": "<actionEn step2>", "result": "<expectedResultEn step2>"}
      ]
    }
  }' \
  https://xray.cloud.getxray.app/api/v2/graphql
```

成功返回：
```json
{
  "data": {
    "createTest": {
      "test": {
        "issueId": "12345",
        "jira": {"key": "PROJ-123"}
      },
      "warnings": []
    }
  }
}
```

注意：
- steps 在 Xray 中直接存储为结构化数据（不在 description 里）
- description 字段可选，如果为空不要传
- jira.fields.description 只放 descriptionEn，不要放 wiki markup 的步骤表格

### 4.4 创建 Precondition

```bash
PRECONDITION_ISSUE_TYPE_ID="10200"

curl -s -X POST \
  -H "Authorization: Bearer $XRAY_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "mutation createPrecondition($jira: JSON!, $definition: String) { createPrecondition(preconditionType: { name: \"Manual\" }, definition: $definition, jira: $jira) { precondition { issueId jira(fields: [\"key\"]) } warnings } }",
    "variables": {
      "jira": {
        "fields": {
          "summary": "<precondition line text>",
          "project": {"id": "'"$PROJECT_ID"'"},
          "issuetype": {"id": "'"$PRECONDITION_ISSUE_TYPE_ID"'"}
        }
      },
      "definition": "<precondition line text>"
    }
  }' \
  https://xray.cloud.getxray.app/api/v2/graphql
```

每行前置条件创建一个 Precondition issue。summary 取该行文本（换行替换为空格）。

### 4.5 关联 Precondition 到 Test

```bash
TEST_ISSUE_ID="12345"        # 从 4.3 响应的 issueId
PRECOND_ISSUE_IDS="[\"11111\",\"11112\"]"  # 从 4.4 响应收集

curl -s -X POST \
  -H "Authorization: Bearer $XRAY_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "mutation addPreconditionsToTest($issueId: String!, $preconditionIssueIds: [String]!) { addPreconditionsToTest(issueId: $issueId, preconditionIssueIds: $preconditionIssueIds) { addedPreconditions warning } }",
    "variables": {
      "issueId": "'"$TEST_ISSUE_ID"'",
      "preconditionIssueIds": '"$PRECOND_ISSUE_IDS"'
    }
  }' \
  https://xray.cloud.getxray.app/api/v2/graphql
```

## 5. 错误处理

### 标准 JIRA 错误

```json
{
  "errorMessages": ["..."],
  "errors": {"field": "error message"}
}
```

### Xray GraphQL 错误

```json
{
  "errors": [{"message": "error description"}]
}
```

### 常见错误及处理

| 错误 | 原因 | 处理 |
|------|------|------|
| 400 Bad Request | 字段缺失或格式错误 | 检查 projectKey、issueType 是否正确 |
| 401 Unauthorized | 凭证错误 | 检查 username + apiToken |
| 404 Not Found | 项目不存在 | 检查 projectKey 拼写 |
| Xray auth failed | clientId/clientSecret 错误 | 检查 Xray 凭证 |
| "Test" issue type not found | Xray 未在项目中启用 | 在 JIRA 项目设置中启用 Xray |
| Priority name not found | 优先级名称不匹配 | 用 ID 替代名称，或省略 priority 字段 |
