> **Markdown 是记忆，Git 是数据库，目录是状态机，Agent 是执行者。**

---

# 1. 架构设计

## Agent职责

```text
Planner
│
├─ 分析需求
├─ 维护架构文档
├─ 拆解任务
├─ 分配任务（移动到 coding/）
├─ 决定任务优先级
└─ 处理需求变更 / BLOCKED任务

Coder
│
├─ 读取任务
├─ 编写代码
├─ 编写测试
└─ 移动任务到 review/

Reviewer
│
├─ 检查任务完成度
├─ 检查架构符合性
├─ 检查代码质量
└─ 输出Review结果
```

---

# 2. docs目录结构

```text
docs/

├── requirements.md       # 项目总目标
├── process.md            # 本工作流说明（即本文档）
├── conventions.md        # 编码规范
├── decisions.md          # 架构决策记录

├── prompt/
│   ├── planner-prompt.md
│   ├── coder-prompt.md
│   ├── reviewer-prompt.md
│   └── orchestrator-prompt.md

├── architecture/
│   ├── system.md
│   ├── backend.md
│   ├── frontend.md
│   └── database.md

├── requirements/
│   ├── RQ-001-user.md
│   ├── RQ-002-role.md
│   └── RQ-003-auth.md

├── tasks/
│   ├── todo/             # 等待领取
│   ├── coding/           # 开发中（Planner分配后移入）
│   ├── review/           # 待审查
│   ├── blocked/          # 被阻塞（需Planner介入）
│   └── done/             # 已完成

└── reviews/              # Review结果存档
```

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

每个业务需求一个文件。

```text
requirements/
  RQ-001-user.md
  RQ-002-role.md
  RQ-003-auth.md
```

格式：

```markdown
# 登录认证

需求：用户名密码登录

验收：
- 返回JWT
- 支持刷新Token
- 支持退出登录
```

---

# 5. Task规范

由 Planner 生成，放入 `tasks/todo/`。

```text
tasks/todo/TASK-001-login.md
```

格式：

```markdown
# TASK-001

Title: 实现登录接口

Requirement: RQ-003-auth

Dependencies: TASK-000（若有前置任务）

目标: 实现JWT登录

验收标准:
- POST /login
- 返回JWT
- 返回RefreshToken

涉及模块:
- AuthController
- AuthService

优先级: HIGH

FailCount: 0
```

> `FailCount` 由 Reviewer 在每次 FAIL 时 +1，达到 3 次自动转为 BLOCKED。

---

# 6. Review规范

Review结果独立保存，文件名含轮次编号。

```text
reviews/TASK-001-review-1.md
```

格式：

```markdown
# TASK-001 Review-1

Result: FAIL

问题:
1. RefreshToken未设置过期时间
2. 缺少单元测试

建议:
- 补充Token配置
- 增加测试
```

> Coder 重做时，读取 `reviews/` 目录下编号最大的 review 文件。

---

# 7. 状态机

状态由文件所在目录决定。

```text
正常流程：
  todo → coding → review → done

FAIL（FailCount < 3）：
  review → coding

FAIL（FailCount ≥ 3）：
  review → blocked

BLOCKED：
  blocked → （Planner修改需求/架构后）→ todo
```

> BLOCKED 出口：Planner 修改相关 requirements 或 architecture，
> 在 Task 文件中注明处理结论，然后将 Task 移回 `todo/`，FailCount 清零。

---

# 8. Planner工作流程

启动后读取：

```text
requirements.md
requirements/*
architecture/*
decisions.md
tasks/*
```

然后：

```text
1. 寻找未完成需求

2. 检查 tasks/todo/ 中 Dependencies 已满足的任务

3. 将就绪任务移动到 tasks/coding/（分配给Coder）

4. 处理 tasks/blocked/（修改需求/架构后移回 todo/）

5. 若无可分配任务，生成新 TASK 放入 todo/
```

---

## Planner Prompt模板

```text
你是Planner Agent。

职责：
1. 阅读 requirements / architecture / decisions
2. 检查已完成任务，寻找未覆盖需求
3. 处理 blocked 任务

规则：
- 每个Task预计30分钟内完成
- 单Task不超过10个文件修改
- 单Task聚焦一个功能
- Task有Dependencies时，前置Task必须在done/才可分配
- 同时最多分配3个Task到coding/（防止并发冲突）

输出：TASK-XXX.md
```

---

# 9. Coder工作流程

读取：

```text
architecture/*
conventions.md
tasks/coding/TASK-XXX.md
reviews/TASK-XXX-review-N.md（若存在，取最大N）
```

开发完成：

```text
tasks/coding/ → tasks/review/
```

---

## Coder Prompt模板

```text
你是Coder Agent。

职责：根据Task实现代码。

规则：
- 遵守 architecture 和 conventions
- 不允许修改 requirements 和 architecture
- 若存在Review记录，优先修复其中问题

完成后：
- 输出变更说明
- 将任务从 coding/ 移动到 review/
```

---

# 10. Reviewer工作流程

读取：

```text
TASK文件
architecture/*
git diff
```

检查：

```text
功能正确性 / 架构符合性 / 测试完整性
```

输出到 `reviews/TASK-XXX-review-N.md`（N自增）。

更新 Task 文件中的 `FailCount`（FAIL时+1）。

---

## Reviewer Prompt模板

```text
你是Reviewer Agent。

检查：
1. 是否满足Task验收标准
2. 是否符合Architecture
3. 是否符合Conventions
4. 是否存在明显Bug

结果：
- PASS    → 任务移到 done/
- FAIL    → FailCount+1；若FailCount<3，移回 coding/；否则移到 blocked/
- BLOCKED → 任务移到 blocked/，说明冲突原因
```

---

# 11. 正常通过案例

```text
Planner：RQ-003-auth → TASK-001 → tasks/todo/
Planner：依赖满足，移动到 tasks/coding/
Coder：实现代码 → tasks/review/
Reviewer：PASS → reviews/TASK-001-review-1.md → tasks/done/
```

---

# 12. FAIL案例

```text
Reviewer：FAIL，FailCount=1 → reviews/TASK-001-review-1.md → tasks/coding/
Coder：读取 review-1，修复 → tasks/review/
Reviewer：PASS → tasks/done/
```

FAIL达到上限：

```text
Reviewer：FAIL，FailCount=3 → tasks/blocked/
Planner：重新拆分或修改需求 → FailCount清零 → tasks/todo/
```

---

# 13. BLOCKED案例

Reviewer发现需求与架构冲突：

```text
Requirement: 登录失败返回200
Architecture: REST规范返回401
```

```text
Reviewer：BLOCKED → tasks/blocked/（说明冲突）
Planner：修改 requirements 或 architecture，在Task中注明结论
Planner：FailCount清零，移回 tasks/todo/
```

---

# 14. 需求变更案例

开发中新增需求：

```text
支持短信登录
```

处理规则：

1. 新增 `requirements/RQ-004-sms-login.md`
2. 不修改旧Task
3. 检查 `coding/` 和 `review/` 中是否有受影响的Task
   - 有冲突 → 移入 `blocked/`，Planner评估后重新分配
   - 无冲突 → 继续原流程
4. Planner生成新Task：TASK-008、TASK-009 → `tasks/todo/`

---

# 15. 最佳实践

每个Agent执行前：

```text
/clear → 重新读取 docs/ + 当前Task
```

不依赖聊天历史。

```text
Markdown = 真相
Git      = 历史
Context  = 临时缓存
```

这样无论换 Claude Code、Cursor、GPT Agent，或 Agent 崩溃，整个项目都能从文件中恢复，继续运转。
