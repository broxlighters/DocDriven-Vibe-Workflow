# 飞书多维表格配置

## 环境变量

```bash
export LARK_APP_TOKEN=<多维表格的 app_token>   # 打开多维表格，URL 中 /base/ 后的字符串
export TASKS_TABLE_ID=<任务表的 table_id>       # 右键表格标签 → 复制链接获取
```

建议写入项目根目录的 `.env`（已加入 .gitignore）。

## 任务表字段结构

| 字段名 | 类型 | 选项 / 说明 |
|--------|------|-------------|
| TaskID | 文本 | TASK-001 |
| Title | 文本 | 任务标题 |
| Status | 单选 | todo / coding / review / done / blocked |
| Requirement | 文本 | RQ-XXX |
| Dependencies | 文本 | 前置 TASK 编号，逗号分隔，无则留空 |
| Priority | 单选 | HIGH / MEDIUM / LOW |
| FailCount | 数字 | 默认 0 |

## 常用命令

```bash
# 按状态查询任务
lark-cli base record list --app-token $LARK_APP_TOKEN --table-id $TASKS_TABLE_ID \
  --filter 'CurrentValue.[Status]="coding"'

# 更新任务状态
lark-cli base record update --app-token $LARK_APP_TOKEN --table-id $TASKS_TABLE_ID \
  --record-id <record_id> --fields '{"Status":"review"}'

# 同时更新多个字段
lark-cli base record update --app-token $LARK_APP_TOKEN --table-id $TASKS_TABLE_ID \
  --record-id <record_id> --fields '{"Status":"coding","FailCount":1}'

# 创建新记录
lark-cli base record create --app-token $LARK_APP_TOKEN --table-id $TASKS_TABLE_ID \
  --fields '{"TaskID":"TASK-001","Title":"...","Status":"todo","Priority":"HIGH","FailCount":0}'
```

> 命令参数以 `lark-cli base record --help` 为准。
