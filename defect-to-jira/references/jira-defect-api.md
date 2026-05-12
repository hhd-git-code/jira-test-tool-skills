# JIRA Defect API Quick Reference

所有命令使用 curl，凭证从 `~/.claude/defect-to-jira-config.json` 读取。

## 通用变量

从配置文件读取后设置为环境变量，避免在命令行暴露敏感信息：

```bash
SERVER_URL="https://company.atlassian.net"
USERNAME="user@company.com"
API_TOKEN="your-api-token"
PROJECT_KEY="PROJ"
ISSUE_TYPE="Bug"
```

## 1. 连接测试

```bash
curl -s -w "\n%{http_code}" \
  -u "$USERNAME:$API_TOKEN" \
  "$SERVER_URL/rest/api/3/myself"
```

成功返回 200 + 用户信息 JSON。

## 2. 获取优先级列表

如果创建缺陷时优先级名称不被 JIRA 接受（400 错误），需获取实际可用优先级：

```bash
curl -s \
  -u "$USERNAME:$API_TOKEN" \
  "$SERVER_URL/rest/api/3/priority"
```

返回 JSON 数组，每个元素有 `id` 和 `name`。必要时用 ID 替代名称：`"priority": {"id": "3"}`。

## 3. 获取项目可用 Issue Type

```bash
curl -s \
  -u "$USERNAME:$API_TOKEN" \
  "$SERVER_URL/rest/api/3/project/$PROJECT_KEY"
```

返回项目信息，`issueTypes` 数组列出所有可用的 issue type 名称和 ID。

## 4. 创建 JIRA 缺陷 Issue

**重要：创建 Issue 必须使用 v2 端点**，因为 v3 只接受 ADF (Atlassian Document Format) 作为 description，不支持 wiki markup。

```bash
curl -s -X POST \
  -u "$USERNAME:$API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "fields": {
      "project": {"key": "'"$PROJECT_KEY"'"},
      "summary": "<summaryEn>",
      "description": "<wikiMarkupDescription>",
      "issuetype": {"name": "'"$ISSUE_TYPE"'"},
      "priority": {"name": "<Priority>"}
    }
  }' \
  "$SERVER_URL/rest/api/2/issue"
```

成功返回：`{"id":"12345","key":"PROJ-123","self":"https://..."}`

失败返回：`{"errorMessages":[],"errors":{"summary":"...","priority":"..."}}`

### Wiki Markup Description 格式（FROZEN，不可修改）

每个字段用 `{noformat}` 块包裹。字段为空时也保留 section 和空的 `{noformat}` 块。

```
h2. Timestamp\n{noformat}<timestampEn>{noformat}\n\nh2. Precondition\n{noformat}<preconditionEn>{noformat}\n\nh2. Steps\n{noformat}<stepsEn>{noformat}\n\nh2. Expected Result\n{noformat}<expectedResultEn>{noformat}\n\nh2. Actual Result\n{noformat}<actualResultEn>{noformat}\n\nh2. Reproduce Rate\n{noformat}<reproduceRateEn>{noformat}\n\nh2. Recover Steps\n{noformat}<recoverStepsEn>{noformat}
```

注意事项：
- wiki markup 中换行用 `\n`
- JSON 字符串中双引号需转义为 `\"`
- 换行符需转义为 `\\n`（在 JSON 字符串内）
- summary 字段有长度限制（通常 255 字符）

### 完整 curl 创建示例

```bash
# 1. 从配置读取凭证
CONFIG=$(cat ~/.claude/defect-to-jira-config.json)
SERVER_URL=$(echo "$CONFIG" | python3 -c "import sys,json; print(json.load(sys.stdin)['jiraDefect']['serverUrl'])")
USERNAME=$(echo "$CONFIG" | python3 -c "import sys,json; print(json.load(sys.stdin)['jiraDefect']['username'])")
API_TOKEN=$(echo "$CONFIG" | python3 -c "import sys,json; print(json.load(sys.stdin)['jiraDefect']['apiToken'])")
PROJECT_KEY=$(echo "$CONFIG" | python3 -c "import sys,json; print(json.load(sys.stdin)['jiraDefect']['projectKey'])")
ISSUE_TYPE=$(echo "$CONFIG" | python3 -c "import sys,json; print(json.load(sys.stdin)['jiraDefect']['issueType'])")

# 2. 构造 description
DESCRIPTION="h2. Timestamp\n{noformat}${TIMESTAMP_EN}{noformat}\n\nh2. Precondition\n{noformat}${PRECONDITION_EN}{noformat}\n\nh2. Steps\n{noformat}${STEPS_EN}{noformat}\n\nh2. Expected Result\n{noformat}${EXPECTED_EN}{noformat}\n\nh2. Actual Result\n{noformat}${ACTUAL_EN}{noformat}\n\nh2. Reproduce Rate\n{noformat}${REPRODUCE_RATE_EN}{noformat}\n\nh2. Recover Steps\n{noformat}${RECOVER_STEPS_EN}{noformat}"

# 3. 创建 Issue
curl -s -X POST \
  -u "$USERNAME:$API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "fields": {
      "project": {"key": "'"$PROJECT_KEY"'"},
      "summary": "'"$SUMMARY_EN"'",
      "description": "'"$DESCRIPTION"'",
      "issuetype": {"name": "'"$ISSUE_TYPE"'"},
      "priority": {"name": "'"$PRIORITY"'"}
    }
  }' \
  "$SERVER_URL/rest/api/2/issue"
```

## 5. 上传附件

```bash
ISSUE_KEY="PROJ-123"
FILE_PATH="/path/to/screenshot.png"

curl -s -X POST \
  -u "$USERNAME:$API_TOKEN" \
  -H "X-Atlassian-Token: no-check" \
  -F "file=@$FILE_PATH" \
  "$SERVER_URL/rest/api/2/issue/$ISSUE_KEY/attachments"
```

关键：
- 必须设置 `X-Atlassian-Token: no-check` header，否则 JIRA 会拒绝附件上传（XSRF 保护）
- 多文件上传：重复 `-F "file=@$PATH"` 参数

```bash
curl -s -X POST \
  -u "$USERNAME:$API_TOKEN" \
  -H "X-Atlassian-Token: no-check" \
  -F "file=@/path/to/screenshot.png" \
  -F "file=@/path/to/trace.log" \
  -F "file=@/path/to/video.mp4" \
  "$SERVER_URL/rest/api/2/issue/$ISSUE_KEY/attachments"
```

返回 JSON 数组，每个元素包含 `id`、`filename`、`content`（下载 URL）。

## 6. 优先级名称映射

标准 JIRA 优先级（名称因 JIRA 实例不同而异，需要动态获取）：

**默认映射（SKILL.md 中的选项值）→ JIRA 实际优先级**：

| SKILL 选项 | 中文 | JIRA 默认名称 | JIRA Cloud 名称 | 映射策略 |
|------------|------|---------------|----------------|----------|
| Blocker | 阻塞 | Blocker | Highest | 优先用 ID |
| Critical | 严重 | Critical | High | 优先用 ID |
| Major | 重要 | Major | Medium | 优先用 ID |
| Minor | 一般 | Minor | Low | 优先用 ID |
| Trivial | 轻微 | Trivial | Lowest | 优先用 ID |

**最佳实践**：创建缺陷时，**优先级用 ID 而非名称**，因为名称因 JIRA 实例而异。首次配置后自动获取优先级列表存储在配置中。

中文→英文自动映射：阻塞→Blocker、严重→Critical、重要→Major、一般→Minor、轻微→Trivial

如果用名称创建失败，用 Step 2 获取实际列表并使用 ID：`"priority": {"id": "3"}`。

## 7. 错误处理

| 错误 | HTTP 状态码 | 处理 |
|------|-------------|------|
| 凭证错误 | 401 | 检查 username + apiToken，建议重新配置 |
| 项目不存在 | 404 | 检查 projectKey |
| 字段缺失/格式错误 | 400 | 检查 summary、issuetype、JSON 格式 |
| 优先级名称不存在 | 400 | 获取优先级列表，用 ID 替代或调整名称 |
| Issue Type 不存在 | 400 | 获取项目可用 issueType 列表 |
| 附件上传失败 | 403 | 检查 X-Atlassian-Token header |
