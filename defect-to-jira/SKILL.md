---
name: defect-to-jira
description: 创建 JIRA 缺陷（Bug），支持单条和批量（Excel），自动翻译中文到英文。当用户提到"创建缺陷"、"提Bug"、"报缺陷"、"JIRA Bug"、"defect"、"批量缺陷"、"Excel缺陷"、"缺陷模板"时使用。
---

你是专业的软件测试工程师，擅长创建 JIRA 缺陷，支持中英文自动翻译。

## 工作流程

严格按以下步骤执行。每步用户确认后才进入下一步。

### Step 0: 经验查找

执行前自动调用 experience-lookup skill，搜索关键词："JIRA缺陷"、"缺陷创建"、"翻译"、"Excel批量"，获取过往经验。

### Step 1: 配置检查

1. 用 Bash 执行 `cat ~/.claude/defect-to-jira-config.json 2>/dev/null` 读取配置文件（不要用 Read 工具，避免缓存问题）
2. 判断逻辑：
   - 如果命令输出为空（文件不存在或内容为空），进入首次配置流程
   - 如果输出非空，用 `python3 -c "import json,sys; d=json.load(sys.stdin); assert all(d['jiraDefect'].get(k) for k in ['serverUrl','username','apiToken','projectKey'])"` 验证 JSON 格式和必填字段
   - 验证失败（JSON 解析错误或缺少必填字段），也进入首次配置流程
3. 配置文件不存在或无效时，进入首次配置流程：
   a. 通过 AskUserQuestion 逐项收集配置（详细流程见 [references/template-system.md](references/template-system.md) 中的首次配置流程）
   b. 必须收集：serverUrl、username、apiToken、projectKey、issueType（默认 Bug）
   c. customDict 可选
4. 验证 JIRA 连接（使用 v3 端点，v2 的 /myself 不接受邮箱认证）：
   ```bash
   curl -s -o /dev/null -w "%{http_code}" -u "$USERNAME:$API_TOKEN" "$SERVER_URL/rest/api/3/myself"
   ```
   期望返回 200
5. **验证通过后，自动获取 JIRA 优先级列表**并存储到配置中（用 python3 urllib 请求，避免 curl -u 邮箱问题）：
   - 请求 `$SERVER_URL/rest/api/3/priority`，提取每个优先级的 id 和 name
   - 存入配置文件的 `priorityMap` 字段，格式：`[{"id": "1", "name": "Highest"}, ...]`
   - 创建缺陷时用 `priority.id` 而非 `priority.name`，避免名称不匹配
6. **用 Write 工具将收集到的配置写入 `~/.claude/defect-to-jira-config.json`**。验证失败不写入，提示用户修正
7. 用户说"更新配置"、"重新配置"时，重新运行设置流程

### Step 2: 选择模式

AskUserQuestion：
- 创建单条缺陷
- 从 Excel 批量创建
- 管理缺陷模板

### Step 3A: 单条缺陷创建

#### 3A.1 模板选择（可选）

1. 从配置文件读取 templates 数组
2. 如果有模板，AskUserQuestion 询问是否使用模板
3. 如果选择使用模板，列出模板名称让用户选择
4. 应用模板：将模板中非空字段作为默认值，用户后续输入可覆盖

#### 3A.2 收集缺陷信息

分 3 轮 AskUserQuestion（每轮最多 4 个问题）：

**第 1 轮**：
- 问题 1："缺陷标题（Summary）" - 文本输入
- 问题 2："优先级（Priority）" - 选项：Blocker / Critical / Major / Minor / Trivial
- 问题 3："时间点（Timestamp）" - 文本输入
- 问题 4："前提条件（Precondition）" - 文本输入

**第 2 轮**：
- 问题 1："复现步骤（Steps）" - 文本输入（多行内容用 \\n 表示换行）
- 问题 2："预期结果（Expected Result）" - 文本输入
- 问题 3："实际结果（Actual Result）" - 文本输入

**第 3 轮**：
- 问题 1："复现率（Reproduce Rate）" - 选项：Always / Often / Sometimes / Rarely / Unable to Reproduce
- 问题 2："恢复步骤（Recover Steps）" - 文本输入
- 问题 3："附件文件路径（可选，多个用逗号分隔，输入'无'跳过）" - 文本输入

如果用户选择了模板，已填写的字段显示当前值，用户可以直接确认保留或修改。

#### 3A.3 翻译

1. 读取配置文件中的 customDict
2. 将所有中文内容翻译为英文：
   - customDict 中的术语**必须**按词典翻译（最长匹配优先）
   - 其他内容由 Claude 翻译，保持技术术语一致性
   - Priority 值保持英文原文
   - ReproduceRate 值保持英文原文
3. 中文优先级名称自动映射：阻塞→Blocker、严重→Critical、重要→Major、一般→Minor、轻微→Trivial
4. 中文复现率自动映射：总是→Always、经常→Often、有时→Sometimes、很少→Rarely、无法复现→Unable to Reproduce
5. 翻译风格：专业、简洁、符合软件缺陷报告英语习惯

#### 3A.4 确认预览

向用户展示翻译预览（中英对照表）：

```
| 字段 | 中文原文 | 英文翻译 |
|------|----------|----------|
| Summary | ... | ... |
| Priority | ... | ... |
| ... | ... | ... |
```

- AskUserQuestion：确认创建 / 修改内容 / 取消

#### 3A.5 附件验证

用户在第 3 轮 AskUserQuestion 中已输入附件路径，此步骤验证路径有效性：

1. 如果用户输入"无"或"跳过"或留空，则无附件，跳过此步骤
2. 解析用户输入：
   - 多个路径用逗号分隔
   - 去除路径首尾空格和可能的引号
3. 验证每个路径是否存在（`ls -la "$FILE_PATH"`）：
   - 如果是文件：加入附件列表
   - 如果是目录：列出目录内容，AskUserQuestion 让用户选择上传哪些文件
   - 如果路径不存在：提示用户检查，跳过该路径
4. 输出附件列表确认

#### 3A.6 创建 JIRA 缺陷

**所有 curl 命令模板见 [references/jira-defect-api.md](references/jira-defect-api.md)**

1. 从配置文件读取凭证，设置为环境变量
2. 构造 JIRA wiki markup description（格式**不可修改**，见下方 FROZEN 格式）
3. **优先级映射**：从配置文件的 `priorityMap` 读取 ID 映射。将用户选择的优先级名称映射为优先级 ID：
   - Blocker → priorityMap 第 1 个的 id（通常 "1"，名称为 Highest）
   - Critical → priorityMap 第 2 个的 id（通常 "2"，名称为 High）
   - Major → priorityMap 第 3 个的 id（通常 "3"，名称为 Medium）
   - Minor → priorityMap 第 4 个的 id（通常 "4"，名称为 Low）
   - Trivial → priorityMap 第 5 个的 id（通常 "5"，名称为 Lowest）
   - 创建时使用 `"priority": {"id": "<mapped_id>"}` 而非 `"priority": {"name": "..."}`，避免名称不匹配
4. 用 python3 urllib 创建 Issue（避免 curl -u 对邮箱中 @ 符号的解析问题）：
   ```python
   import json, urllib.request, base64
   # 从配置读取凭证，构造 Basic Auth header
   payload = {"fields": {"project": {"key": PROJECT_KEY}, "summary": summaryEn, "description": wikiMarkup, "issuetype": {"name": ISSUE_TYPE}, "priority": {"id": PRIORITY_ID}}}
   req = urllib.request.Request(f"{SERVER_URL}/rest/api/2/issue", data=json.dumps(payload).encode(), headers={"Authorization": f"Basic {encoded}", "Content-Type": "application/json"}, method="POST")
   resp = urllib.request.urlopen(req)
   result = json.loads(resp.read().decode())
   issue_key = result["key"]
   ```
5. 解析响应，提取 `key`
6. 如果创建失败（非 201），输出错误信息并停止
7. 如果有附件，逐个上传（附件上传用 curl 或 python3 均可）：
   ```bash
   curl -s -X POST -u "$USERNAME:$API_TOKEN" -H "X-Atlassian-Token: no-check" -F "file=@$FILE_PATH" "$SERVER_URL/rest/api/2/issue/$ISSUE_KEY/attachments"
   ```
8. 附件上传失败不阻塞，记录失败但继续上传其余附件
9. 输出结果：成功返回 `[OK] 缺陷已创建: PROJ-123`，链接为 `$SERVER_URL/browse/PROJ-123`

#### FROZEN 描述格式（不可修改）

```
h2. Timestamp
{noformat}{timestampEn}{noformat}

h2. Precondition
{noformat}{preconditionEn}{noformat}

h2. Steps
{noformat}{stepsEn}{noformat}

h2. Expected Result
{noformat}{expectedResultEn}{noformat}

h2. Actual Result
{noformat}{actualResultEn}{noformat}

h2. Reproduce Rate
{noformat}{reproduceRateEn}{noformat}

h2. Recover Steps
{noformat}{recoverStepsEn}{noformat}
```

字段为空时保留 section 标题和空的 `{noformat}` 块。

#### 3A.7 保存模板（可选）

1. AskUserQuestion 询问是否保存为模板
2. 是 → AskUserQuestion 输入模板名称
3. 检查名称不重复
4. 将当前缺陷的 9 个字段值写入配置文件的 templates 数组
5. 否 → 结束

### Step 3B: Excel 批量创建

#### 3B.1 获取 Excel 文件

AskUserQuestion 询问 Excel/CSV 文件路径

#### 3B.2 解析文件

1. 用 Bash 检查 `python3 -c "import openpyxl" 2>/dev/null` 是否可用
2. 不可用时执行 `pip3 install --user openpyxl`
3. 将 Python 解析脚本写入 `/tmp/defect-parse-excel.py`（脚本见 [references/excel-format.md](references/excel-format.md)）
4. 用 Bash 执行 `python3 /tmp/defect-parse-excel.py "$FILE_PATH"`
5. 解析输出 JSON，提取 `items`、`unmappedColumns`、`totalRows`
6. 清理临时文件 `rm /tmp/defect-parse-excel.py`
7. 如果 `unmappedColumns` 不为空，列出未识别的列名

#### 3B.3 预览确认

1. 输出解析结果预览：总行数、识别的列名、未识别的列名
2. 显示前 5 行数据摘要（编号 + summary + priority）
3. AskUserQuestion：全部创建 / 只创建部分 / 取消
4. 如果"只创建部分"，AskUserQuestion 询问要创建哪些行号（如 "1, 3, 5-8"）

#### 3B.4 必填项检查与补全

1. 检查所有待创建缺陷的必填字段：summary、steps、expectedResult、actualResult 不能为空
2. 识别空字段，输出缺失清单，如：
   ```
   第 1 行「导航打不开」: 缺少 steps, expectedResult, actualResult
   第 3 行「QQ音乐打不开」: 缺少 steps, expectedResult, actualResult
   ```
3. 如果存在空字段，AskUserQuestion 询问如何处理：
   - 逐条补全：对每条缺少必填项的缺陷，用 AskUserQuestion 逐轮收集缺失字段（规则同 3A.2，每轮最多 4 个问题）
   - 跳过空字段：保留为空，直接创建（description 中空字段保留空的 `{noformat}` 块）
   - 跳过这些行：不创建缺少必填项的缺陷
4. 选择"逐条补全"时，对每条缺陷按缺失字段分轮收集：
   - 如缺 1-4 个字段，1 轮完成
   - 如缺 5-8 个字段，分 2 轮完成
   - 已有字段显示当前值，用户确认保留或修改

#### 3B.5 批量翻译和创建

对每条缺陷：
1. 翻译中文内容为英文（同 3A.3 翻译规则，应用 customDict + Claude 翻译）
2. 构造 wiki markup description（FROZEN 格式）
3. 通过 curl 创建 JIRA Issue
4. 立即输出进度：`[OK] 第 1/N 条: PROJ-123` 或 `[FAIL] 第 1/N 条: <error>`
5. 单条失败不阻塞后续创建

#### 3B.6 报告结果

1. 输出汇总：成功 N/M 条，列出失败的行号和原因
2. 写入 `jira-defect-creation-report.md`，包含每个缺陷的：
   - 行号、标题（中文+英文）
   - JIRA Issue Key、Issue 链接
   - 失败项的错误信息
3. 如果有失败项，AskUserQuestion 询问是否重试

### Step 3C: 模板管理

详细操作见 [references/template-system.md](references/template-system.md)

支持的操作：
- 列出所有模板（名称 + 非空字段摘要）
- 新建模板（从当前缺陷值 / 从头填写）
- 编辑模板
- 删除模板

### 错误处理

| 错误 | 处理 |
|------|------|
| 401 Unauthorized | 提示凭证错误，建议运行"更新配置" |
| 404 Not Found | 检查 projectKey 是否正确 |
| Priority name not found | 用 GET /rest/api/3/priority 获取可用列表，或用 ID |
| Issue Type not found | 用 GET /rest/api/3/project/$PROJECT_KEY 获取可用列表 |
| JIRA Cloud v2 /myself 返回 401 | 用 /rest/api/3/myself 验证连接，v2 /issue 端点仍用于创建（支持 wiki markup） |
| 文件路径不存在 | 提示用户检查路径，跳过该附件 |
| openpyxl 安装失败 | 提示用户手动安装或将 Excel 另存为 CSV |
| 单条创建失败不阻塞后续 | 收集所有失败后统一报告 |

## 注意事项

- 所有中间文件写入当前工作目录
- curl 命令中的 JSON 需要正确转义特殊字符（双引号 `\"`、换行 `\\n`、反斜杠 `\\`）
- 不要在命令行参数中暴露 API Token，优先用环境变量传递（从配置文件读取后 export）
- description 的 wiki markup 格式是 FROZEN 的，严禁修改
- 优先级和复现率支持中文输入，自动映射为英文
- 用户提供的文件路径可能包含空格，curl -F "file=@$PATH" 中路径需要加引号
