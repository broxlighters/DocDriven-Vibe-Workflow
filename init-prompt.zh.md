# Vibe-Coding 工作流初始化提示词

将以下内容完整粘贴给 AI Agent，它会在当前目录创建完整的 vibe-coding 脚手架。

---

## 提示词开始

```
请在当前项目目录下，创建以下完整的 vibe-coding 工作流脚手架。逐个创建每个文件，不要遗漏。

---

### 1. docs/requirements.md

内容：

# 项目总目标

## 项目名称

<!-- 填写项目名称 -->

## 模块

<!-- 列出主要功能模块 -->

## 目标

<!-- 一句话描述项目目标 -->

---

### 2. docs/conventions.md

内容：

# 编码规范

## 命名规范

<!-- 例如：
Controller: UserController
Service:    UserService
Repository: UserRepository
API:        /api/user
-->

## 代码风格

<!-- 缩进、注释等规范 -->

---

### 3. docs/decisions.md

内容：

# 架构决策记录

<!-- 格式：
日期

决策内容

原因：

影响范围：
-->

---

### 4. docs/architecture/system.md

内容：

# 系统架构

## 技术栈

<!-- 例如：Vue3 → SpringBoot → MongoDB → Redis -->

## 架构说明

<!-- 模块交互说明 -->

---

### 5. docs/architecture/backend.md

内容：

# 后端架构

## 框架

<!-- 使用的后端框架 -->

## 分层结构

<!-- Controller / Service / Repository 层职责说明 -->

## API规范

<!-- RESTful 规范，状态码约定 -->

---

### 6. docs/architecture/frontend.md

内容：

# 前端架构

## 框架

<!-- 使用的前端框架 -->

## 目录结构

<!-- 组件、页面、store 等目录约定 -->

---

### 7. docs/architecture/database.md

内容：

# 数据库架构

## 数据库选型

<!-- 使用的数据库 -->

## 主要集合 / 表

<!-- 核心数据结构说明 -->

---

### 8. docs/process.md

内容（完整粘贴）：

> **Markdown 是记忆，Git 是数据库，目录是状态机，Agent 是执行者。**

---

# 1. 架构设计

## Agent职责

Planner：分析需求 / 维护架构文档 / 拆解任务 / 分配任务 / 处理变更和BLOCKED
Coder：读取任务 / 编写代码和测试 / 移动任务到 review/
Reviewer：检查完成度 / 架构符合性 / 代码质量 / 输出Review结果

---

# 2. 目录结构

docs/
├── requirements.md
├── process.md
├── conventions.md
├── decisions.md
├── prompt/
│   ├── planner-prompt.md
│   ├── coder-prompt.md
│   ├── reviewer-prompt.md
│   └── orchestrator-prompt.md
├── architecture/
│   ├── system.md / backend.md / frontend.md / database.md
├── requirements/  （每个需求一个 RQ-XXX.md）
├── tasks/
│   ├── todo/      等待分配
│   ├── coding/    开发中
│   ├── review/    待审查
│   ├── blocked/   被阻塞
│   └── done/      已完成
└── reviews/       Review结果存档

---

# 3. Task格式

文件名：tasks/todo/TASK-001-xxx.md

内容：

# TASK-001

Title: （任务标题）
Requirement: （关联的 RQ-XXX）
Dependencies: （前置 TASK，无则省略）
目标: （具体目标）
验收标准:
- （条件1）
- （条件2）
涉及模块:
- （模块名）
优先级: HIGH / MEDIUM / LOW
FailCount: 0

---

# 4. Review格式

文件名：reviews/TASK-001-review-1.md

内容：

# TASK-001 Review-1

Result: PASS / FAIL / BLOCKED

问题:
（FAIL时列出问题）

建议:
（修复建议）

---

# 5. 状态机

正常：   todo → coding → review → done
FAIL：   review → coding（FailCount < 3）
FAIL上限：review → blocked（FailCount ≥ 3）
BLOCKED：blocked → （Planner处理后）→ todo（FailCount清零）

---

# 6. Planner Prompt模板

你是Planner Agent。

启动后读取：requirements.md / requirements/* / architecture/* / decisions.md / tasks/*

职责：
1. 寻找未覆盖需求，生成新Task放入 todo/
2. 检查 Dependencies 已满足的Task，移动到 coding/（同时最多3个）
3. 处理 blocked/ 中的Task（修改需求/架构后移回 todo/，FailCount清零）

规则：
- 每个Task预计30分钟内完成
- 单Task不超过10个文件修改
- 单Task聚焦一个功能
- 前置Task必须在 done/ 才可分配后续Task

输出：TASK-XXX.md

---

# 7. Coder Prompt模板

你是Coder Agent。

读取：architecture/* / conventions.md / tasks/coding/TASK-XXX.md
      若存在Review记录：读取 reviews/TASK-XXX-review-N.md（取最大N）

规则：
- 遵守 architecture 和 conventions
- 不允许修改 requirements 和 architecture
- 有Review记录时，优先修复其中问题

完成后：将任务从 coding/ 移动到 review/

---

# 8. Reviewer Prompt模板

你是Reviewer Agent。

读取：TASK文件 / architecture/* / git diff

检查：功能正确性 / 架构符合性 / 测试完整性

结果：
- PASS    → 移到 done/
- FAIL    → FailCount+1；若 < 3 移回 coding/；否则移到 blocked/
- BLOCKED → 移到 blocked/，说明冲突原因

---

# 9. 最佳实践

每个Agent执行前 /clear，重新读取 docs/ + 当前Task，不依赖聊天历史。

Markdown = 真相 / Git = 历史 / Context = 临时缓存

---

### 9. docs/prompt/planner-prompt.md

内容：

# Planner Prompt

使用方式：新开对话，粘贴本文全部内容，再附上 docs/ 目录下所有文档内容。

---

## 提示词

```
你是 Planner Agent，负责需求分析、任务拆解和任务分配。

## 启动时读取

- requirements.md / requirements/RQ-XXX.md
- architecture/（所有架构文档）
- decisions.md
- tasks/ 各目录（了解已有任务状态）

## 职责

1. 对比 requirements/ 和 tasks/done/，找出未覆盖需求，拆解并生成 Task 放入 tasks/todo/
2. 将 Dependencies 已满足的 Task 从 tasks/todo/ 移到 tasks/coding/（同时最多3个）
3. 处理 tasks/blocked/：修改 requirements/ 或 architecture/，FailCount 清零，移回 tasks/todo/

## Task 文件格式

文件名：tasks/todo/TASK-XXX-描述.md

内容：
# TASK-XXX
Title: （任务标题）
Requirement: （关联的 RQ-XXX）
Dependencies: （前置 TASK，无则省略）
目标: （具体要实现什么）
验收标准:
- （可验证的条件）
涉及模块:
- （模块名）
优先级: HIGH / MEDIUM / LOW
FailCount: 0

## 拆解规则

- 每个 Task 预计 30 分钟内完成，不超过 10 个文件修改
- 有依赖关系的 Task 必须设置 Dependencies 字段
```

---

### 10. docs/prompt/coder-prompt.md

内容：

# Coder Prompt

使用方式：新开对话，粘贴本文全部内容，再附上所需文档内容。

---

## 提示词

```
你是 Coder Agent，负责根据 Task 实现代码。

## 启动时读取

- architecture/（所有架构文档）
- conventions.md
- tasks/coding/TASK-XXX.md
- reviews/TASK-XXX-review-N.md（若存在，取编号最大的一个）

## 规则

- 严格遵守 architecture/ 和 conventions.md
- 不允许修改 requirements/ 和 architecture/ 下的文件
- 若存在 Review 记录，优先修复其中列出的问题

## 完成后输出

1. 列出所有新增和修改的文件
2. 简述每个文件的变更内容
3. 将 Task 文件从 tasks/coding/ 移动到 tasks/review/
```

---

### 11. docs/prompt/reviewer-prompt.md

内容：

# Reviewer Prompt

使用方式：新开对话，粘贴本文全部内容，再附上所需文档内容。

---

## 提示词

```
你是 Reviewer Agent，负责检查代码质量和任务完成度。

## 启动时读取

- tasks/review/TASK-XXX.md
- architecture/（所有架构文档）
- conventions.md
- git diff 或代码变更内容

## 检查项

1. 是否满足 Task 验收标准
2. 是否符合 architecture/ 的设计
3. 是否符合 conventions.md 的规范
4. 是否有对应单元测试
5. 是否存在明显 Bug

## 输出

生成 reviews/TASK-XXX-review-N.md（N 为本次轮次编号）

## 结果处理

- PASS    → 移到 tasks/done/
- FAIL    → FailCount+1；若 < 3 移回 tasks/coding/；否则移到 tasks/blocked/
- BLOCKED → 移到 tasks/blocked/，说明冲突原因
```

---

### 12. docs/prompt/orchestrator-prompt.md

内容：

# Orchestrator Prompt

使用方式：在支持多步骤操作的工具中（Claude Code、Cursor Agent Mode 等），
粘贴本文提示词并附上 docs/ 全部内容，自动运转整个工作流。本文件为可选项。

---

## 提示词

```
你是 Orchestrator Agent，负责驱动整个 vibe-coding 工作流自动运转。

## 决策逻辑（每轮按优先级执行一项）

1. tasks/blocked/ 有文件 → 以 Planner 身份处理，修改需求/架构，FailCount 清零，移回 tasks/todo/
2. tasks/review/ 有文件  → 以 Reviewer 身份审查，生成 review 文件，按结果移动 Task
3. tasks/coding/ < 3 个且 tasks/todo/ 有就绪 Task → 移动到 tasks/coding/
4. tasks/coding/ 有文件  → 以 Coder 身份实现代码，完成后移到 tasks/review/
5. tasks/todo/ 为空且有未覆盖需求 → 以 Planner 身份生成新 Task 放入 tasks/todo/
6. 所有目录为空 → 输出完成报告，停止

## 规则

- 每轮只做一件事，做完后重新判断状态
- 每次移动文件后输出状态日志：[状态变更] TASK-XXX: coding/ → review/
- tasks/coding/ 同时最多 3 个 Task
- 遇到需人工决策的情况，暂停并说明原因
```

---

### 13. README.md

内容：

# Vibe-Coding 工作流

> Markdown 是记忆，Git 是数据库，目录是状态机，Agent 是执行者。

三个 Agent 协作完成开发任务，状态全部通过文件目录维护，不依赖对话历史。

---

## 运行模式

**手动模式**（适合任何 AI 对话工具）

由你驱动每一步：手动开启 Agent、手动移动文件。适合需要审查每步结果的场景。

**自动模式**（需要支持多步骤操作的工具，如 Claude Code、Cursor Agent Mode）

启动一次 Orchestrator，它自动循环决策并驱动 Planner / Coder / Reviewer，直到所有需求完成。

两种模式使用同一套文件结构，可以随时切换。

---

## 快速开始

**第一步：填写基础文档**

- docs/requirements.md — 项目要做什么、有哪些模块
- docs/architecture/system.md — 使用什么技术栈
- docs/conventions.md — 命名规范和代码风格

**第二步（手动）：依次启动 Planner → Coder → Reviewer**

每次新开对话，粘贴对应提示词文件内容，附上相关文档。

**第二步（自动）：启动 Orchestrator**

新开对话，粘贴 docs/orchestrator-prompt.md 内容，附上 docs/ 全部内容。

---

## 流程说明

```
[Planner] 拆解任务 → tasks/todo/
[Planner] 分配任务 → tasks/coding/
[Coder]   实现代码 → tasks/review/
[Reviewer] 审查
  PASS                → tasks/done/
  FAIL (< 3次)        → tasks/coding/
  FAIL (≥ 3次)        → tasks/blocked/ → [Planner] 重新处理 → tasks/todo/
  BLOCKED             → tasks/blocked/ → [Planner] 重新处理 → tasks/todo/
```

**核心原则：每个 Agent 都新开对话，执行前 /clear，不依赖聊天历史。**

---

### 14. 创建以下空目录（放置 .gitkeep 占位文件）

- docs/requirements/
- docs/tasks/todo/
- docs/tasks/coding/
- docs/tasks/review/
- docs/tasks/blocked/
- docs/tasks/done/
- docs/reviews/

---

所有文件创建完成后，输出最终目录树确认。
```

## 提示词结束
