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
export LARK_PROFILE=<lark-cli profile 名>           # 应用/机器人身份所在 profile（如 cli-bot），供 --profile 引用，勿写死
export LARK_APP_ID=<应用 App ID>                    # 如 cli_xxx；供预检自举 config init
export LARK_APP_SECRET=<应用 App Secret>            # 飞书后台「凭证与基础信息」复制；供预检自举，跨用户/沙箱自动恢复 bot 身份
```

建议写入项目根目录的 `.env`（已加入 .gitignore）。

## 身份与登录预检（每次执行任何 lark-cli base 命令前）

**所有 `lark-cli base` 命令统一用应用（机器人）身份 TAT，免扫码**：每条命令都追加
`--as bot --profile "$LARK_PROFILE"`（`LARK_PROFILE` 从 `.env` 读取，**不要写死 profile 名**）。

TAT（tenant_access_token，应用身份）用 app_id + app_secret 直接换取，**无需用户扫码授权、无 7 天续期上限**，由 lark-cli 自动获取与刷新。相比 UAT（用户身份，要扫码、refresh token 7 天过期），更适合 agent / CI 长期无人值守运行。

> **跨用户 / 沙箱要点**：lark-cli 的 app secret 存在「当前 Windows 用户」的凭据库里（DPAPI 加密），**换一个系统用户（如 Codex 沙箱账户 `*\codexsandboxoffline`）就解不开**，表现为 `bot=not_configured`，即便 `profile list` 能列出该 profile。解法：把 `LARK_APP_ID` / `LARK_APP_SECRET` 放进 `.env`，预检时若 bot 未就绪，就用它们 `config init` **自举**——在当前用户的凭据库里补写凭据，从而自动恢复。

```bash
# 先加载 .env 拿到 LARK_PROFILE / LARK_APP_ID / LARK_APP_SECRET 等
set -a; source .env; set +a

# 取 bot 身份状态。auth status 只支持 --json（无 --format/--jq），用 python 取字段
bot_status() {
  lark-cli auth status --json --profile "$LARK_PROFILE" 2>/dev/null \
    | python -c "import sys,json;print(json.load(sys.stdin)['identities']['bot']['status'])" 2>/dev/null
}
BOT=$(bot_status)

# bot 未就绪 → 用 .env 凭据自举（跨用户/沙箱自动恢复），再复检一次
if [ "$BOT" != "ready" ]; then
  if [ -n "$LARK_APP_ID" ] && [ -n "$LARK_APP_SECRET" ]; then
    printf '%s' "$LARK_APP_SECRET" | lark-cli config init \
      --name "$LARK_PROFILE" --app-id "$LARK_APP_ID" --app-secret-stdin --brand feishu >/dev/null 2>&1
    BOT=$(bot_status)
  fi
fi

if [ "$BOT" != "ready" ]; then
  echo "应用身份仍未就绪：检查 .env 的 LARK_PROFILE / LARK_APP_ID / LARK_APP_SECRET 是否正确" >&2
fi
```

> **为什么用 TAT 而非 UAT**：UAT 每次（或 token 过期后）都要用户扫码，agent 无人值守时反复中断；
> TAT 是应用自身凭据，配好一次后**永久免扫码**、自动续期。代价：
> - 记录的「创建人/修改人」显示为机器人，不是某个真人；
> - 需在飞书开发者后台为该 app **申请对应 base scope**（如 `base:record:read/create/update`、`base:table:read`、`base:field:read`），并**发布版本**；
> - 需把该机器人**加为目标多维表格的协作者**（文档级授权），否则即便有 scope 也读不到数据。
>
> 易错点：`auth status` 不支持 `--format`/`--jq`，只支持 `--json` 和 `--verify`；
> `base +record-list` 等子命令则**支持** `--format json --jq`。两者相反，别混用。

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

字段可用 `lark-cli base +field-create` 创建。`single_select` 也可写成 `{"type":"select","multiple":false,...}`。`--json` 同样走 `@文件`（理由见下节「JSON 参数」）：

```bash
# 单选字段
cat > ./.lark_tmp.json <<'EOF'
{"name":"Status","type":"single_select","property":{"options":[{"name":"TODO"},{"name":"IN_PROGRESS"},{"name":"DONE"}]}}
EOF
lark-cli base +field-create --as bot --profile $LARK_PROFILE --base-token $LARK_APP_TOKEN --table-id $REQUIREMENTS_TABLE_ID \
  --json @./.lark_tmp.json --format json
rm -f ./.lark_tmp.json

# 多行文本字段
cat > ./.lark_tmp.json <<'EOF'
{"name":"Description","type":"text"}
EOF
lark-cli base +field-create --as bot --profile $LARK_PROFILE --base-token $LARK_APP_TOKEN --table-id $REQUIREMENTS_TABLE_ID \
  --json @./.lark_tmp.json --format json
rm -f ./.lark_tmp.json

# 双向关联字段（Tasks 表的 Requirement 指向 Requirements 表）
# heredoc 不加引号，$REQUIREMENTS_TABLE_ID 在写文件时展开成字面 table_id
cat > ./.lark_tmp.json <<EOF
{"name":"Requirement","type":"link","property":{"table_id":"$REQUIREMENTS_TABLE_ID"}}
EOF
lark-cli base +field-create --as bot --profile $LARK_PROFILE --base-token $LARK_APP_TOKEN --table-id $TASKS_TABLE_ID \
  --json @./.lark_tmp.json --format json
rm -f ./.lark_tmp.json
```

> `link` 字段创建后，被指向的表（Requirements）会自动获得反向关联列。

## 常用命令

> 命令以本机 `lark-cli base --help` 为准。读取用 `+record-list`，
> 创建/更新统一用 `+record-upsert`（**无 `--record-id` 创建，有 `--record-id` 更新**）。
> `--json` 是顶层字段映射，**不要再包一层 `fields`**。
>
> **身份（重要）**：下面所有 `lark-cli base` 示例为简洁省略了身份参数，**实际执行时每条都要追加**
> `--as bot --profile "$LARK_PROFILE"`（先 `set -a; source .env; set +a` 加载）。例如读取：
> ```bash
> lark-cli base +record-list --base-token "$LARK_APP_TOKEN" --table-id "$TASKS_TABLE_ID" \
>   --as bot --profile "$LARK_PROFILE" --format json
> ```

### JSON 参数：`--json` 与 `--filter-json` 一律用 `@文件`，不要内联（避免 PowerShell 转义坑）

凡是带 JSON 值的参数——**写入用的 `--json`** 和 **查询用的 `--filter-json`**——下面示例里写成内联 `'{...}'` 只是**为了展示 JSON 长什么样**。**实际执行时（尤其在 Windows PowerShell 下）不要内联**——PowerShell 会剥掉 JSON 里的双引号、按空格把参数拆开，导致写入失败、静默不创建记录，或查询第一次必失败后再回退。统一改用 lark-cli 原生支持的 `@文件` 语法。约定两个临时文件名，避免同一流程里查询与写入互相覆盖：**写入用 `./.lark_tmp.json`，查询用 `./.lark_filter.json`**。

```bash
# —— 写入（--json）——
# 1. 把 JSON 写到【项目目录下】的临时文件（@file 只接受当前目录的相对路径，不接受绝对路径或 /tmp）
cat > ./.lark_tmp.json <<'EOF'
{"ReqID":"RQ-003-auth","Title":"登录认证 含空格","Status":"TODO","Description":"支持 \"引号\" 与 空格"}
EOF
# 2. 用 @相对路径 传参（传的是文件名，零转义，bash / PowerShell 通用）
lark-cli base +record-upsert --base-token "$LARK_APP_TOKEN" --table-id "$REQUIREMENTS_TABLE_ID" \
  --as bot --profile "$LARK_PROFILE" --json @./.lark_tmp.json --format json
# 3. 用完即删
rm -f ./.lark_tmp.json

# —— 查询（--filter-json）——
cat > ./.lark_filter.json <<'EOF'
{"logic":"and","conditions":[["Status","==","coding"]]}
EOF
lark-cli base +record-list --base-token "$LARK_APP_TOKEN" --table-id "$TASKS_TABLE_ID" \
  --as bot --profile "$LARK_PROFILE" --filter-json @./.lark_filter.json --format json
rm -f ./.lark_filter.json
```

> 要点：① 文件**必须在项目目录内**、用 `./` 相对路径（`@/tmp/x.json` 或 `@C:\...` 会被拒：*--file must be a relative path within the current directory*）；② 删除类高危命令（如 `+record-delete`）需加 `--yes` 才会执行；③ `--json` 仍是顶层字段映射，不要包 `fields`；④ 临时文件均已加入 .gitignore（`.lark_*.json`）。

### 读取（`--filter-json` 走 `@./.lark_filter.json`）

```bash
# 按状态查询任务
cat > ./.lark_filter.json <<'EOF'
{"logic":"and","conditions":[["Status","==","coding"]]}
EOF
lark-cli base +record-list --as bot --profile $LARK_PROFILE --base-token $LARK_APP_TOKEN --table-id $TASKS_TABLE_ID \
  --filter-json @./.lark_filter.json --format json
rm -f ./.lark_filter.json

# 按 ReqID 查需求记录（拿 record_id 用于关联）
cat > ./.lark_filter.json <<'EOF'
{"logic":"and","conditions":[["ReqID","==","RQ-003-auth"]]}
EOF
lark-cli base +record-list --as bot --profile $LARK_PROFILE --base-token $LARK_APP_TOKEN --table-id $REQUIREMENTS_TABLE_ID \
  --filter-json @./.lark_filter.json --format json --jq '.data.items[0].record_id'
rm -f ./.lark_filter.json
```

### 创建需求记录（Requirements 表）

```bash
cat > ./.lark_tmp.json <<'EOF'
{"ReqID":"RQ-003-auth","Title":"登录认证","Status":"TODO","Priority":"MVP","Description":"用户名密码登录","Features":"用户名密码登录\n刷新 Token\n退出登录","AcceptanceCriteria":"返回 JWT\n支持刷新 Token\n支持退出登录"}
EOF
lark-cli base +record-upsert --as bot --profile $LARK_PROFILE --base-token $LARK_APP_TOKEN --table-id $REQUIREMENTS_TABLE_ID \
  --json @./.lark_tmp.json --format json
rm -f ./.lark_tmp.json
```

### 创建任务记录（Tasks 表，含 Requirement 关联）

关联字段值是 record_id 数组，形如 `[{"id":"rec_xxx"}]`，需先查到所属 RQ 的 record_id：

```bash
# 1. 查 RQ 的 record_id
cat > ./.lark_filter.json <<'EOF'
{"logic":"and","conditions":[["ReqID","==","RQ-003-auth"]]}
EOF
REQ_REC=$(lark-cli base +record-list --as bot --profile $LARK_PROFILE --base-token $LARK_APP_TOKEN --table-id $REQUIREMENTS_TABLE_ID \
  --filter-json @./.lark_filter.json --format json --jq '.data.items[0].record_id')
rm -f ./.lark_filter.json

# 2. 创建 Task，Requirement 写关联（$REQ_REC 在写文件时展开，文件里就是字面 record_id）
cat > ./.lark_tmp.json <<EOF
{"TaskID":"TASK-001","Title":"实现登录接口","Status":"todo","Requirement":[{"id":"$REQ_REC"}],"Dependencies":"","Priority":"HIGH","FailCount":0,"Goal":"实现 JWT 登录","AcceptanceCriteria":"POST /login\n返回 JWT\n返回 RefreshToken","Modules":"AuthController\nAuthService"}
EOF
lark-cli base +record-upsert --as bot --profile $LARK_PROFILE --base-token $LARK_APP_TOKEN --table-id $TASKS_TABLE_ID \
  --json @./.lark_tmp.json --format json
rm -f ./.lark_tmp.json
```

> 注意第 2 步 heredoc 用的是不带引号的 `EOF`（`<<EOF` 而非 `<<'EOF'`），这样 `$REQ_REC` 才会展开成真实 record_id；JSON 里没有其它 `$`，所以安全。

### 更新记录（带 `--record-id`）

```bash
# 更新任务状态
cat > ./.lark_tmp.json <<'EOF'
{"Status":"review"}
EOF
lark-cli base +record-upsert --as bot --profile $LARK_PROFILE --base-token $LARK_APP_TOKEN --table-id $TASKS_TABLE_ID \
  --record-id <record_id> --json @./.lark_tmp.json --format json
rm -f ./.lark_tmp.json

# 同时更新多个字段（Reviewer 一次写入 Review 结果 + 状态）
cat > ./.lark_tmp.json <<'EOF'
{"Status":"coding","FailCount":1,"ReviewResult":"FAIL","ReviewRound":1,"ReviewProblems":"RefreshToken 未设过期\n缺少单元测试","ReviewSuggestions":"补充 Token 配置\n增加测试"}
EOF
lark-cli base +record-upsert --as bot --profile $LARK_PROFILE --base-token $LARK_APP_TOKEN --table-id $TASKS_TABLE_ID \
  --record-id <record_id> --json @./.lark_tmp.json --format json
rm -f ./.lark_tmp.json

# 更新需求状态
cat > ./.lark_tmp.json <<'EOF'
{"Status":"DONE"}
EOF
lark-cli base +record-upsert --as bot --profile $LARK_PROFILE --base-token $LARK_APP_TOKEN --table-id $REQUIREMENTS_TABLE_ID \
  --record-id <record_id> --json @./.lark_tmp.json --format json
rm -f ./.lark_tmp.json
```

> 多行文本字段用 `\n` 分隔多条内容。
> 单选值直接写选项名（如 `"todo"`、`"PASS"`），关联字段写 `[{"id":"rec_xxx"}]`。
