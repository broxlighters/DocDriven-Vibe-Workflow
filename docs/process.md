> **Markdown 是记忆，Git 是数据库，飞书多维表格是状态机，Agent 是执行者。**

> 需求和任务的状态统一由飞书多维表格（Lark Base）的两张表管理：
> **Requirements 表**（每行一个 RQ）与 **Tasks 表**（每行一个 Task，通过双向关联指向 RQ）。
> Review 结果（最新一轮）存在 Task 同一行，不再有 `requirements/RQ-XXX.md` 和 `reviews/` 文件。
> Base 配置见 `docs/lark-base.md`。

---

# 1. 架构设计

## Agent职责

```text
Orchestrator
│
├─ 查询 Base 各状态任务
├─ 按优先级判断当前该做什么
├─ 以对应 Agent 身份执行一步
└─ 循环驱动，直到所有需求完成

Analyst
│
├─ 与用户对话收集需求
├─ 整理模块、功能点、优先级
├─ 确认 MVP 范围
└─ 输出 requirements.md，并在 Requirements 表创建 RQ 记录

Planner
│
├─ 分析需求（读 Requirements 表）
├─ 维护架构文档
├─ 拆解任务（创建 Tasks 记录，Requirement 关联到 RQ）
├─ 分配任务（Status → coding）
├─ 维护 RQ 的完成状态（更新 Requirements 表 Status）
└─ 处理需求变更 / BLOCKED任务

Coder
│
├─ 读取 coding 任务
├─ 编写代码
├─ 编写测试
└─ 更新任务 Status → review

Reviewer
│
├─ 检查任务完成度
├─ 检查架构符合性
├─ 检查代码质量
└─ 把 Review 结果写入 Task 同一行，更新任务 Status
```

---

# 2. docs目录结构

```text
docs/

├── requirements.md       # 项目总目标
├── process.md            # 本工作流说明（即本文档）
├── conventions.md        # 编码规范
├── decisions.md          # 架构决策记录
├── lark-base.md          # 飞书多维表格配置

├── prompt/
│   ├── orchestrator-prompt.md
│   ├── analyst-prompt.md
│   ├── planner-prompt.md
│   ├── coder-prompt.md
│   └── reviewer-prompt.md

└── architecture/
    ├── system.md
    ├── backend.md
    ├── frontend.md
    └── database.md
```

> 逐条需求（RQ）没有本地文件——存在 Requirements 表。
> Task 没有本地文件——其全部信息（含目标、验收标准、涉及模块、最新一轮 Review）都存在 Tasks 表的记录字段中，状态由 `Status` 字段表达。

---

# 3. 每个文件的职责

## requirements.md

项目总目标

```markdown
# 权限管理平台

模块：用户管理 / 角色管理 / 权限管理 / 登录认证

目标：统一管理企业权限
```

---

## architecture/system.md

系统架构

```markdown
Vue3 → SpringBoot → MongoDB → Redis
```

---

## conventions.md

编码规范

```markdown
Controller: UserController
Service:     UserService
Repository:  UserRepository
API:         /api/user
```

---

## decisions.md

架构决策记录

```markdown
2026-06-18

采用MongoDB

原因：权限结构变化频繁

影响范围：database.md / 所有Repository层
```

---

# 4. Requirement管理

每个业务需求一条 **Requirements 表记录**（不再有 `requirements/RQ-XXX.md` 文件）。

字段（详见 `docs/lark-base.md`）：

| 字段 | 说明 |
|------|------|
| ReqID | RQ-001-user |
| Title | 模块 / 需求名称 |
| Status | TODO / IN_PROGRESS / DONE |
| Priority | MVP / 迭代 |
| Description | 需求职责简述 |
| Features | 功能点，每行一条 |
| AcceptanceCriteria | 验收点，每行一条 |
| Tasks | 双向关联，反向显示该 RQ 的所有 Task |

## Requirement 完成判定

一个 RQ 会被 Planner 拆成多个 Task，**只有该 RQ 关联的所有 Task 都完成，RQ 才算完成**。

RQ 记录实现后**不删除**——它是需求的永久记忆，Planner 寻找未完成需求、Coder/Reviewer 通过 Task 的 `Requirement` 关联反查它，都依赖它留在表里。完成与否通过 `Status` 字段表达，由 Planner 维护：

```text
TODO         尚未拆解，或验收点尚未被 Task 完整覆盖
IN_PROGRESS  已拆出 Task，但仍有 Task 未完成
DONE         已完整拆解，且关联的所有 Task 在 Base 中均为 Status=done
```

> 关键前提：仅凭「已拆的 Task 都 done」不足以判定 DONE，必须先确认该 RQ 的全部验收点都已拆成 Task，否则漏拆的功能点会被误判为完成。

Planner 判定时按 `Requirement` 关联查询 Tasks 表（也可读 RQ 记录的反向 `Tasks` 字段）：

```bash
lark-cli base +record-list --base-token $LARK_APP_TOKEN --table-id $TASKS_TABLE_ID \
  --filter-json '{"logic":"and","conditions":[["Requirement","==","RQ-XXX"]]}'
```

---

# 5. Task规范

由 Planner 生成：直接创建一条 Tasks 表记录，Task 的全部信息都写进字段，**没有本地文件**。`Requirement` 是双向关联字段，需先查到所属 RQ 的 record_id 再写入。

Tasks 表记录字段（详见 `docs/lark-base.md`）：

| 字段 | 说明 |
|------|------|
| TaskID | TASK-001 |
| Title | 任务标题 |
| Status | todo / coding / review / done / blocked |
| Requirement | 双向关联，指向所属 RQ |
| Dependencies | 前置 TASK 编号，逗号分隔 |
| Priority | HIGH / MEDIUM / LOW |
| FailCount | 默认 0 |
| Goal | 具体要实现什么 |
| AcceptanceCriteria | 验收标准，多条用 `\n` 分隔 |
| Modules | 涉及的模块/文件，多个用 `\n` 分隔 |
| ReviewResult / ReviewRound / ReviewProblems / ReviewSuggestions | 最新一轮 Review 结果 |

```bash
# 先查所属 RQ 的 record_id
REQ_REC=$(lark-cli base +record-list --base-token $LARK_APP_TOKEN --table-id $REQUIREMENTS_TABLE_ID \
  --filter-json '{"logic":"and","conditions":[["ReqID","==","RQ-003-auth"]]}' \
  --format json --jq '.data.items[0].record_id')

lark-cli base +record-upsert --base-token $LARK_APP_TOKEN --table-id $TASKS_TABLE_ID \
  --json '{"TaskID":"TASK-001","Title":"实现登录接口","Status":"todo","Requirement":[{"id":"'"$REQ_REC"'"}],"Dependencies":"","Priority":"HIGH","FailCount":0,"Goal":"实现JWT登录","AcceptanceCriteria":"POST /login\n返回JWT\n返回RefreshToken","Modules":"AuthController\nAuthService"}'
```

> `FailCount` 由 Reviewer 在每次 FAIL 时 +1，达到 3 次自动转为 blocked。

---

# 6. Review规范

Review 结果写进 **Task 同一行** 的 Review 字段，**只保留最新一轮**（不再有 `reviews/TASK-XXX-review-N.md` 文件）。历史轮次由 `FailCount` / `ReviewRound` 体现。

| 字段 | 说明 |
|------|------|
| ReviewResult | PASS / FAIL / BLOCKED |
| ReviewRound | 本轮审查编号 |
| ReviewProblems | 问题列表，每行一条（FAIL/BLOCKED 时填，PASS 留空） |
| ReviewSuggestions | 修复建议，供 Coder 参考 |

Reviewer 用**一条命令**同时写 Review 字段和 Status（示例为 FAIL）：

```bash
lark-cli base +record-upsert --base-token $LARK_APP_TOKEN --table-id $TASKS_TABLE_ID \
  --record-id <record_id> \
  --json '{"Status":"coding","FailCount":1,"ReviewResult":"FAIL","ReviewRound":1,"ReviewProblems":"RefreshToken未设置过期时间\n缺少单元测试","ReviewSuggestions":"补充Token配置\n增加测试"}'
```

> Coder 重做时，直接读 Task 记录的 `ReviewProblems` / `ReviewSuggestions` 字段。

---

# 7. 状态机

状态由 Tasks 表记录的 `Status` 字段决定，状态流转通过 `lark-cli base +record-upsert --record-id` 完成。

```text
正常流程：
  todo → coding → review → done

FAIL（FailCount < 3）：
  review → coding（FailCount +1）

FAIL（FailCount ≥ 3）：
  review → blocked

BLOCKED：
  blocked → （Planner修改需求/架构后）→ todo（FailCount 清零）
```

状态更新示例：

```bash
lark-cli base +record-upsert --base-token $LARK_APP_TOKEN --table-id $TASKS_TABLE_ID \
  --record-id <record_id> --json '{"Status":"review"}'
```

> BLOCKED 出口：Planner 修改相关 requirements 或 architecture，
> 在 Task 记录的 `ReviewSuggestions`（或 decisions.md）中注明处理结论，
> 然后将记录 Status 改回 todo，FailCount 清零。

---

# 8. Orchestrator工作流程

Orchestrator 驱动整个循环，每轮按优先级判断当前该做什么，并以对应 Agent 身份执行一步。

启动后读取 `docs/` 下所有文件，并查询 Base 各状态任务数量：

```bash
lark-cli base +record-list --base-token $LARK_APP_TOKEN --table-id $TASKS_TABLE_ID
```

决策优先级（每轮只做一件事，做完重新判断）：

```text
0. Requirements 表为空      → 以 Analyst 身份收集需求
1. Status=blocked 有记录    → 以 Planner 身份处理，清零 FailCount，改回 todo
2. Status=review 有记录     → 以 Reviewer 身份审查，写 Review 字段并按结果更新 Status
3. coding 数 < 3 且 todo 有就绪记录 → 分配：Status → coding
4. Status=coding 有记录     → 以 Coder 身份实现代码，完成后 Status → review
5. todo 为空且有未覆盖需求   → 以 Planner 身份生成新 Task
6. todo/coding/review/blocked 均无记录 → 所有需求完成，输出报告，停止
```

详见 `docs/prompt/orchestrator-prompt.md`。

---

# 9. Analyst工作流程

启动后：

```text
1. 每次只问用户一个问题，逐步收集：
   - 项目目标和目标用户
   - 核心功能模块
   - 各模块的具体功能和边界条件
   - MVP 范围与迭代计划
   - 技术偏好或约束

2. 整理信息，展示结构化摘要供用户确认

3. 用户确认后：
   - 写入 docs/requirements.md（项目总目标）
   - 在 Requirements 表为每个模块创建一条 RQ 记录（Status=TODO）
   - 若用户提供了技术栈，写入 docs/architecture/system.md
```

详见 `docs/prompt/analyst-prompt.md`。

---

# 10. Planner工作流程

启动后读取：

```text
requirements.md
architecture/*
decisions.md
查询 Requirements 表全部 RQ 记录
查询 Tasks 表全部任务记录
```

然后：

```text
1. 维护 RQ 状态：按 Requirement 关联查询每个 RQ 的关联 Task，
   更新 Requirements 表记录的 Status（TODO / IN_PROGRESS / DONE）

2. 寻找 Status 为 TODO / IN_PROGRESS 的 RQ，找出未覆盖的需求点

3. 将未覆盖需求拆解为 Task：创建 Tasks 记录（Status=todo，Requirement 关联到 RQ）

4. 检查 Status=todo 中 Dependencies 已满足的任务，更新 Status → coding
   （同时最多保留 3 个 coding）

5. 处理 Status=blocked：修改需求/架构后，Status 改回 todo，FailCount 清零
```

详见 `docs/prompt/planner-prompt.md`。

---

# 11. Coder工作流程

读取：

```text
architecture/*
conventions.md
查询 Status=coding 的记录（字段含目标/验收标准/涉及模块/上一轮 Review 等全部信息）
```

若该记录的 `ReviewResult=FAIL`，先按 `ReviewProblems` / `ReviewSuggestions` 字段修复。

开发完成后，用查询得到的 record_id 更新状态：

```bash
lark-cli base +record-upsert --base-token $LARK_APP_TOKEN --table-id $TASKS_TABLE_ID \
  --record-id <record_id> --json '{"Status":"review"}'
```

详见 `docs/prompt/coder-prompt.md`。

---

# 12. Reviewer工作流程

读取：

```text
查询 Status=review 的记录（字段含目标/验收标准/涉及模块等全部信息）
architecture/*
conventions.md
git diff
```

检查：

```text
功能正确性 / 架构符合性 / 规范符合性 / 测试完整性 / 明显Bug
```

将 Review 结果写入 Task 同一行的 Review 字段，并更新 Status（一条命令完成）：

```text
PASS              → ReviewResult=PASS，Status=done
FAIL（FailCount<3）→ ReviewResult=FAIL，写 ReviewProblems/Suggestions，Status=coding，FailCount+1
FAIL（FailCount≥3）→ ReviewResult=FAIL，Status=blocked
BLOCKED           → ReviewResult=BLOCKED，Status=blocked
```

详见 `docs/prompt/reviewer-prompt.md`。

---

# 13. 正常通过案例

```text
Analyst：与用户对话 → requirements.md + Requirements 表 RQ-003-auth 记录（Status=TODO）
Planner：RQ-003-auth → TASK-001 → 创建 Tasks 记录（Status=todo，Requirement 关联 RQ-003-auth）
Planner：依赖满足，Status → coding
Coder：实现代码，Status → review
Reviewer：PASS → 写 ReviewResult=PASS → Status=done
Planner：RQ-003 全部 Task 已 done → RQ-003 记录 Status → DONE
```

---

# 14. FAIL案例

```text
Reviewer：FAIL，FailCount=1 → 写 ReviewResult=FAIL + 问题/建议 → Status=coding
Coder：读 Task 的 ReviewProblems/ReviewSuggestions，修复 → Status=review
Reviewer：PASS → 写 ReviewResult=PASS → Status=done
```

FAIL达到上限：

```text
Reviewer：FAIL，FailCount=3 → Status=blocked
Planner：重新拆分或修改需求 → FailCount清零 → Status=todo
```

---

# 15. BLOCKED案例

Reviewer发现需求与架构冲突：

```text
Requirement: 登录失败返回200
Architecture: REST规范返回401
```

```text
Reviewer：BLOCKED → 写 ReviewResult=BLOCKED + 冲突说明 → Status=blocked
Planner：修改 requirements 或 architecture，在 Task 的 ReviewSuggestions 或 decisions.md 注明结论
Planner：FailCount清零，Status → todo
```

---

# 16. 需求变更案例

开发中新增需求：

```text
支持短信登录
```

处理规则：

1. 在 Requirements 表新增 `RQ-004-sms-login` 记录（Status=TODO）
2. 不修改旧Task
3. 检查 Tasks 表中 Status=coding / review 是否有受影响的Task
   - 有冲突 → Status 改为 blocked，Planner评估后重新分配
   - 无冲突 → 继续原流程
4. Planner生成新Task：TASK-008、TASK-009 → Tasks 表（Status=todo，Requirement 关联 RQ-004）

---

# 17. 最佳实践

每个Agent执行前：

```text
/clear → 重新读取 docs/ + 查询 Base 当前需求与任务
```

不依赖聊天历史。

```text
Markdown   = 真相（项目总目标 / 架构 / 规范）
Lark Base  = 状态（需求 + 任务 + 审查流转）
Git        = 历史
Context    = 临时缓存
```

这样无论换 Claude Code、Cursor、GPT Agent，或 Agent 崩溃，整个项目都能从文件 + Base 中恢复，继续运转。
