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

### 9. 创建以下空目录（放置 .gitkeep 占位文件）

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
