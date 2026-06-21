# 飞书多维表格配置

整个工作流用 **两张表** 管理状态：

- **Requirements 表**：每行一个需求（RQ），是父记录。
- **Tasks 表**：每行一个 Task，是子记录；通过「双向关联」字段 `Requirement` 指向 Requirements 表。每个 Task 的 Review 结果（最新一轮）也存在同一行。

> Markdown 只保留项目级真相（requirements.md 总目标、architecture、conventions、decisions）。
> **逐条需求、任务、审查结果都在 Base 里**，不再有 `docs/requirements/RQ-XXX.md` 和 `docs/reviews/TASK-XXX-review-N.md` 文件。

## 环境变量

```bash
export LARK_APP_TOKEN=<多维表格的 app_token>        # 打开多维表格，URL 中 /base/ 后的字符串
export REQUIREMENTS_TABLE_ID=<需求表的 table_id>    # 右键表格标签 → 复制链接获取
export TASKS_TABLE_ID=<任务表的 table_id>           # 同上
export LARK_FOLDER_TOKEN=<目标文件夹的 folder_token> # 可选，建 Base 时指定父文件夹，URL 中 /folder/ 后的字符串
```

建议写入项目根目录的 `.env`（已加入 .gitignore）。

## Requirements 表字段结构

| 字段名 | 类型 | 选项 / 说明 |
|--------|------|-------------|
| ReqID | 文本 | RQ-001-user（唯一标识，供关联查找） |
| Title | 文本 | 模块 / 需求名称 |
| Status | 单选 | TODO / IN_PROGRESS / DONE |
| Priority | 单选 | MVP / 迭代 |
| Description | 多行文本 | 该需求的职责简述 |
| Features | 多行文本 | 功能点，每行一条 |
| AcceptanceCriteria | 多行文本 | 验收点，每行一条 |
| Tasks | 双向关联 | 指向 Tasks 表，自动反向显示该 RQ 的所有 Task |

## Tasks 表字段结构

Task 的全部信息（含目标、验收标准、涉及模块、最新一轮 Review）都存在 Base 记录里。

| 字段名 | 类型 | 选项 / 说明 |
|--------|------|-------------|
| TaskID | 文本 | TASK-001 |
| Title | 文本 | 任务标题 |
| Status | 单选 | todo / coding / review / done / blocked |
| Requirement | 双向关联 | 指向 Requirements 表，关联所属 RQ |
| Dependencies | 文本 | 前置 TASK 编号，逗号分隔，无则留空 |
| Priority | 单选 | HIGH / MEDIUM / LOW |
| FailCount | 数字 | 默认 0 |
| Goal | 多行文本 | 具体要实现什么 |
| AcceptanceCriteria | 多行文本 | 验收标准，每行一条 |
| Modules | 多行文本 | 涉及的模块/文件，每行一个 |
| ReviewResult | 单选 | PASS / FAIL / BLOCKED（空 = 尚未审查） |
| ReviewRound | 数字 | 当前审查轮次 |
| ReviewProblems | 多行文本 | 本轮问题列表，每行一条（PASS 时留空） |
| ReviewSuggestions | 多行文本 | 修复建议，供 Coder 参考 |

> Review 只保留**最新一轮**结果，历史轮次由 `FailCount` 体现。

## 创建多维表格（首次配置）

用 `lark-cli base +base-create` 新建一个多维表格。**若配置了 `LARK_FOLDER_TOKEN`，则建到该文件夹下**；不配置则建到「我的空间」根目录。

```bash
lark-cli base +base-create \
  --name "Vibe Coding 工作台" \
  --folder-token $LARK_FOLDER_TOKEN \
  --time-zone Asia/Shanghai
```

> 输出里的 `app_token` 即上文 `LARK_APP_TOKEN` 的值，拿到后再建下面两张表。
> 加 `--dry-run` 可只打印请求不实际执行。

## 一次性建表（首次配置）

字段可用 `lark-cli base +field-create` 创建。`single_select` 也可写成 `{"type":"select","multiple":false,...}`。

```bash
# 单选字段
lark-cli base +field-create --base-token $LARK_APP_TOKEN --table-id $REQUIREMENTS_TABLE_ID \
  --json '{"name":"Status","type":"single_select","property":{"options":[{"name":"TODO"},{"name":"IN_PROGRESS"},{"name":"DONE"}]}}'

# 多行文本字段
lark-cli base +field-create --base-token $LARK_APP_TOKEN --table-id $REQUIREMENTS_TABLE_ID \
  --json '{"name":"Description","type":"text"}'

# 双向关联字段（Tasks 表的 Requirement 指向 Requirements 表）
lark-cli base +field-create --base-token $LARK_APP_TOKEN --table-id $TASKS_TABLE_ID \
  --json '{"name":"Requirement","type":"link","property":{"table_id":"'"$REQUIREMENTS_TABLE_ID"'"}}'
```

> `link` 字段创建后，被指向的表（Requirements）会自动获得反向关联列。

## 常用命令

> 命令以本机 `lark-cli base --help` 为准。读取用 `+record-list`，
> 创建/更新统一用 `+record-upsert`（**无 `--record-id` 创建，有 `--record-id` 更新**）。
> `--json` 是顶层字段映射，**不要再包一层 `fields`**。

### 读取

```bash
# 按状态查询任务
lark-cli base +record-list --base-token $LARK_APP_TOKEN --table-id $TASKS_TABLE_ID \
  --filter-json '{"logic":"and","conditions":[["Status","==","coding"]]}'

# 按 ReqID 查需求记录（拿 record_id 用于关联）
lark-cli base +record-list --base-token $LARK_APP_TOKEN --table-id $REQUIREMENTS_TABLE_ID \
  --filter-json '{"logic":"and","conditions":[["ReqID","==","RQ-003-auth"]]}' \
  --format json --jq '.data.items[0].record_id'
```

### 创建需求记录（Requirements 表）

```bash
lark-cli base +record-upsert --base-token $LARK_APP_TOKEN --table-id $REQUIREMENTS_TABLE_ID \
  --json '{"ReqID":"RQ-003-auth","Title":"登录认证","Status":"TODO","Priority":"MVP","Description":"用户名密码登录","Features":"用户名密码登录\n刷新 Token\n退出登录","AcceptanceCriteria":"返回 JWT\n支持刷新 Token\n支持退出登录"}'
```

### 创建任务记录（Tasks 表，含 Requirement 关联）

关联字段值是 record_id 数组，形如 `[{"id":"rec_xxx"}]`，需先查到所属 RQ 的 record_id：

```bash
# 1. 查 RQ 的 record_id
REQ_REC=$(lark-cli base +record-list --base-token $LARK_APP_TOKEN --table-id $REQUIREMENTS_TABLE_ID \
  --filter-json '{"logic":"and","conditions":[["ReqID","==","RQ-003-auth"]]}' \
  --format json --jq '.data.items[0].record_id')

# 2. 创建 Task，Requirement 写关联
lark-cli base +record-upsert --base-token $LARK_APP_TOKEN --table-id $TASKS_TABLE_ID \
  --json '{"TaskID":"TASK-001","Title":"实现登录接口","Status":"todo","Requirement":[{"id":"'"$REQ_REC"'"}],"Dependencies":"","Priority":"HIGH","FailCount":0,"Goal":"实现 JWT 登录","AcceptanceCriteria":"POST /login\n返回 JWT\n返回 RefreshToken","Modules":"AuthController\nAuthService"}'
```

### 更新记录（带 `--record-id`）

```bash
# 更新任务状态
lark-cli base +record-upsert --base-token $LARK_APP_TOKEN --table-id $TASKS_TABLE_ID \
  --record-id <record_id> --json '{"Status":"review"}'

# 同时更新多个字段（Reviewer 一次写入 Review 结果 + 状态）
lark-cli base +record-upsert --base-token $LARK_APP_TOKEN --table-id $TASKS_TABLE_ID \
  --record-id <record_id> \
  --json '{"Status":"coding","FailCount":1,"ReviewResult":"FAIL","ReviewRound":1,"ReviewProblems":"RefreshToken 未设过期\n缺少单元测试","ReviewSuggestions":"补充 Token 配置\n增加测试"}'

# 更新需求状态
lark-cli base +record-upsert --base-token $LARK_APP_TOKEN --table-id $REQUIREMENTS_TABLE_ID \
  --record-id <record_id> --json '{"Status":"DONE"}'
```

> 多行文本字段用 `\n` 分隔多条内容。
> 单选值直接写选项名（如 `"todo"`、`"PASS"`），关联字段写 `[{"id":"rec_xxx"}]`。
