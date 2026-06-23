# 飞书 Base 操作通用规范（共享）

> 本文件供 analyst / planner / coder / reviewer / orchestrator 各 SKILL 引用。
> 凡是要执行 `lark-cli base` 命令的角色，**动手前先读本文件并完成身份预检**。
> 表配置与字段结构（Requirements 表 / Tasks 表）详见 `docs/lark-base.md`，此处不重复。

## 一、身份预检（先于一切 `lark-cli base` 命令，最高优先）

所有 `lark-cli base` 命令统一用**应用（机器人）身份 TAT**，免扫码：每条命令追加
`--as bot --profile "$LARK_PROFILE"`（profile 名从 `.env` 读，**勿写死**）。
TAT 用 app_id + app_secret 直接换取，无需扫码、无 7 天续期上限，适合 agent 无人值守。

启动第一步，加载 `.env` 并确认该 profile 的应用身份就绪：

```bash
set -a; source .env; set +a   # LARK_PROFILE / LARK_APP_ID / LARK_APP_SECRET / LARK_APP_TOKEN / 两个 TABLE_ID

# auth status 只支持 --json（无 --format/--jq），用 python 取字段
bot_status() {
  lark-cli auth status --json --profile "$LARK_PROFILE" 2>/dev/null \
    | python -c "import sys,json;print(json.load(sys.stdin)['identities']['bot']['status'])" 2>/dev/null
}
BOT=$(bot_status)

# bot 未就绪 → 用 .env 凭据自举（跨用户/沙箱自动恢复），再复检一次
if [ "$BOT" != "ready" ] && [ -n "$LARK_APP_ID" ] && [ -n "$LARK_APP_SECRET" ]; then
  printf '%s' "$LARK_APP_SECRET" | lark-cli config init \
    --name "$LARK_PROFILE" --app-id "$LARK_APP_ID" --app-secret-stdin --brand feishu >/dev/null 2>&1
  BOT=$(bot_status)
fi

if [ "$BOT" != "ready" ]; then
  echo "应用身份仍未就绪：检查 .env 的 LARK_PROFILE / LARK_APP_ID / LARK_APP_SECRET 是否正确" >&2
fi
```

- `BOT=ready` → 应用身份可用，直接进入各角色职责。**TAT 免扫码、自动续期，无需任何登录交互。**
- 仍非 ready → 检查 `.env` 的 `LARK_PROFILE` / `LARK_APP_ID` / `LARK_APP_SECRET`；**不要走扫码登录流程**。

### 常见报错排查

- `bot=not_configured`：多见于**换了系统用户/沙箱账户**（如 `*\codexsandboxoffline`），原凭据库（DPAPI 按当前 Windows 用户加密）解不开。按上面自举脚本用 `LARK_APP_ID`/`LARK_APP_SECRET` `config init` 恢复，再复检；仍失败才提示用户检查 `.env`。**不要走扫码登录。**
- `app_scope_not_applied`（code 99991672）：app 没申请 base scope，提示用户去飞书后台申请对应 scope（`base:record:read/create/update`、`base:table:read`、`base:field:read`）并发布版本。
- `code 1002 "note has been deleted"`：**不是真被删**，是 `LARK_APP_TOKEN` 配错、或机器人未被加为该多维表格协作者。先核对 `.env` 的 `LARK_APP_TOKEN` 与表 URL `/base/` 后那段一致，再确认机器人已是协作者。
- 详见 `docs/lark-base.md`「身份与登录预检」。

## 二、JSON 参数规范：`--json` 与 `--filter-json` 一律走 `@文件`，不要内联

带 JSON 值的参数——**写入用的 `--json`**（如 `+record-upsert`）和 **查询用的 `--filter-json`**——**都不要内联 `'{...}'`**。Windows PowerShell 会剥掉 JSON 里的双引号、按空格拆参，导致写入失败、静默不创建记录，或查询第一次必失败后回退。统一改用 lark-cli 原生 `@文件` 语法。约定两个临时文件名，避免同一流程里查询与写入互相覆盖：

- **写入用 `./.lark_tmp.json`**
- **查询用 `./.lark_filter.json`**

```bash
# —— 写入（--json）——
cat > ./.lark_tmp.json <<'EOF'
{"Status":"review"}
EOF
lark-cli base +record-upsert --as bot --profile "$LARK_PROFILE" --base-token "$LARK_APP_TOKEN" --table-id "$TASKS_TABLE_ID" \
  --record-id <record_id> --json @./.lark_tmp.json --format json
rm -f ./.lark_tmp.json

# —— 查询（--filter-json）——
cat > ./.lark_filter.json <<'EOF'
{"logic":"and","conditions":[["Status","==","coding"]]}
EOF
lark-cli base +record-list --as bot --profile "$LARK_PROFILE" --base-token "$LARK_APP_TOKEN" --table-id "$TASKS_TABLE_ID" \
  --filter-json @./.lark_filter.json --format json
rm -f ./.lark_filter.json
```

要点：
- **JSON 文件必须无 BOM。** lark-cli 的 JSON 解析器拒收开头的 UTF-8 BOM 字节。本文用的 `cat > 文件 <<'EOF'`（bash heredoc）本身就不带 BOM，是推荐写法。**严禁用 PowerShell 5.1 的 `Set-Content -Encoding UTF8` 或 `Out-File -Encoding UTF8`**——它们会写入 BOM 导致写回失败。必须用 PowerShell 时改用无 BOM 写法，例如 `[System.IO.File]::WriteAllText("$PWD\.lark_tmp.json", $json, (New-Object System.Text.UTF8Encoding $false))`，或 PowerShell 7+ 的 `Set-Content -Encoding utf8NoBOM`。
- 文件**必须在项目目录内**、用 `./` 相对路径（`@/tmp/x.json` 或 `@C:\...` 会被拒）。
- 临时文件已在 `.gitignore`（`.lark_*.json`），用完 `rm -f`。
- `--json` 是顶层字段映射，**不要再包一层 `fields`**。
- 删除类高危命令（如 `+record-delete`）需加 `--yes` 才执行。
- 关联字段（如 Task 的 `Requirement`）写 `[{"id":"rec_xxx"}]`，需先查到所属 RQ 的 record_id；**关联字段过滤必须用 record_id，用 `RQ-XXX` 文本过滤会命中 0 条**。
- 占位符（如 `<FailCount+1>`、`<上一轮+1>`）先算出真实数值再写进文件。
- `$REQ_REC` 等变量要展开成字面值时，heredoc 用不带引号的 `<<EOF`；纯静态 JSON 用 `<<'EOF'`。

> 文档里出现的 `--json '{...}'` / `--filter-json '{...}'` 仅为展示 JSON 内容，**实际执行一律按本规范走 `@文件`**。
