# Excel 批量导入格式说明

## Excel 列名映射

支持中文和英文列名，与 jira-test-tool 完全一致：

| 中文列名 | 英文列名 | 映射字段 |
|----------|----------|----------|
| 标题 | Title / Summary | summary |
| 优先级 | Priority | priority |
| 时间点 | Timestamp | timestamp |
| 前提条件 | Precondition | precondition |
| 步骤 | Steps | steps |
| 预期结果 | Expected Result | expectedResult |
| 实际结果 | Actual Result | actualResult |
| 复现率 | Reproduce Rate | reproduceRate |
| Recover步骤 | Recover Steps | recoverSteps |

### 优先级值映射（中文→英文）

| 中文 | 英文 |
|------|------|
| 阻塞 | Blocker |
| 严重 | Critical |
| 重要 | Major |
| 一般 | Minor |
| 轻微 | Trivial |

### 复现率值映射（中文→英文）

| 中文 | 英文 |
|------|------|
| 总是 | Always |
| 经常 | Often |
| 有时 | Sometimes |
| 很少 | Rarely |
| 无法复现 | Unable to Reproduce |

## 文件格式

- **Excel**：`.xlsx` / `.xls`，第一个 sheet，第一行为表头
- **CSV**：UTF-8 编码（支持 BOM），第一行为表头，逗号分隔

## Excel 解析 Python 脚本

skill 通过 Bash 将以下脚本写入 `/tmp/defect-parse-excel.py` 执行：

```python
#!/usr/bin/env python3
"""Parse Excel/CSV file and output JSON for defect-to-jira skill."""
import sys
import json
import os

COLUMN_MAPPING = {
    '标题': 'summary', 'Title': 'summary', 'Summary': 'summary',
    '优先级': 'priority', 'Priority': 'priority',
    '时间点': 'timestamp', 'Timestamp': 'timestamp',
    '前提条件': 'precondition', 'Precondition': 'precondition',
    '步骤': 'steps', 'Steps': 'steps',
    '预期结果': 'expectedResult', 'Expected Result': 'expectedResult',
    '实际结果': 'actualResult', 'Actual Result': 'actualResult',
    '复现率': 'reproduceRate', 'Reproduce Rate': 'reproduceRate',
    'Recover步骤': 'recoverSteps', 'Recover Steps': 'recoverSteps',
}

PRIORITY_ZH_MAP = {
    '阻塞': 'Blocker', '严重': 'Critical', '重要': 'Major',
    '一般': 'Minor', '轻微': 'Trivial',
}

REPRODUCE_RATE_ZH_MAP = {
    '总是': 'Always', '经常': 'Often', '有时': 'Sometimes',
    '很少': 'Rarely', '无法复现': 'Unable to Reproduce',
}

TEXT_FIELDS = {'summary', 'timestamp', 'precondition', 'steps',
               'expectedResult', 'actualResult', 'recoverSteps'}

def map_priority(value):
    if not value:
        return ''
    return PRIORITY_ZH_MAP.get(value, value)

def map_reproduce_rate(value):
    if not value:
        return ''
    mapped = REPRODUCE_RATE_ZH_MAP.get(value)
    if mapped:
        return mapped
    valid = ['Always', 'Often', 'Sometimes', 'Rarely', 'Unable to Reproduce']
    if value in valid:
        return value
    return ''

def process_rows(rows):
    if not rows:
        return {'items': [], 'unmappedColumns': [], 'totalRows': 0}

    headers = list(rows[0].keys())
    field_map = {}
    unmapped = []

    for header in headers:
        trimmed = header.strip()
        if trimmed in COLUMN_MAPPING:
            field_map[header] = COLUMN_MAPPING[trimmed]
        else:
            unmapped.append(trimmed)

    items = []
    for row in rows:
        defect = {f: '' for f in ['summary', 'priority', 'timestamp',
                                    'precondition', 'steps', 'expectedResult',
                                    'actualResult', 'reproduceRate', 'recoverSteps']}
        for header, field in field_map.items():
            value = str(row.get(header, '')).strip()
            if field == 'priority':
                defect[field] = map_priority(value)
            elif field == 'reproduceRate':
                defect[field] = map_reproduce_rate(value)
            elif field in TEXT_FIELDS:
                defect[field] = value
        items.append(defect)

    return {'items': items, 'unmappedColumns': unmapped, 'totalRows': len(rows)}

def main():
    if len(sys.argv) < 2:
        print("Usage: parse-excel.py <file_path>", file=sys.stderr)
        sys.exit(1)

    file_path = sys.argv[1]
    ext = os.path.splitext(file_path)[1].lower()

    if ext == '.csv':
        import csv
        with open(file_path, 'r', encoding='utf-8-sig') as f:
            reader = csv.DictReader(f)
            rows = [row for row in reader if any(v.strip() for v in row.values())]
        result = process_rows(rows)
    else:
        try:
            import openpyxl
        except ImportError:
            import subprocess
            subprocess.check_call([sys.executable, '-m', 'pip', 'install',
                                   '--user', 'openpyxl'],
                                  stdout=subprocess.DEVNULL,
                                  stderr=subprocess.DEVNULL)
            import openpyxl

        wb = openpyxl.load_workbook(file_path, read_only=True, data_only=True)
        ws = wb.active
        rows_list = list(ws.iter_rows(values_only=True))
        wb.close()

        if len(rows_list) < 2:
            result = {'items': [], 'unmappedColumns': [], 'totalRows': 0}
        else:
            headers = [str(h).strip() if h else '' for h in rows_list[0]]
            data_rows = []
            for row in rows_list[1:]:
                row_dict = {}
                for i, header in enumerate(headers):
                    if i < len(row):
                        row_dict[header] = str(row[i]) if row[i] is not None else ''
                    else:
                        row_dict[header] = ''
                if any(v.strip() for v in row_dict.values()):
                    data_rows.append(row_dict)
            result = process_rows(data_rows)

    print(json.dumps(result, ensure_ascii=False))

if __name__ == '__main__':
    main()
```

## Skill 中的调用流程

1. 检查 openpyxl 是否可用：
   ```bash
   python3 -c "import openpyxl" 2>/dev/null && echo "OK" || echo "NEED_INSTALL"
   ```

2. 不可用时自动安装：
   ```bash
   pip3 install --user openpyxl
   ```

3. 将脚本写入临时文件：
   ```bash
   cat > /tmp/defect-parse-excel.py << 'PYEOF'
   [脚本内容]
   PYEOF
   ```

4. 执行解析：
   ```bash
   python3 /tmp/defect-parse-excel.py "$FILE_PATH"
   ```

5. 解析输出 JSON，提取 `items`、`unmappedColumns`、`totalRows`

6. 清理临时文件：
   ```bash
   rm /tmp/defect-parse-excel.py
   ```
