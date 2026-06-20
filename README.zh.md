**中文** | [English](README.md)

# Vibe-Coding 工作流

> Markdown 是记忆，Git 是数据库，目录是状态机，Agent 是执行者。

四个 Agent 协作完成开发任务，状态全部通过文件目录维护，不依赖对话历史。

---

## 目录结构

```
docs/
├── requirements.md          # 项目总目标
├── conventions.md           # 编码规范
├── decisions.md             # 架构决策记录
├── process.md               # 完整工作流说明
├── prompt/
│   ├── analyst-prompt.md        # Analyst 提示词（需求收集）
│   ├── planner-prompt.md        # Planner 提示词
│   ├── coder-prompt.md          # Coder 提示词
│   ├── reviewer-prompt.md       # Reviewer 提示词
│   └── orchestrator-prompt.md   # Orchestrator 提示词（可选，自动驱动全流程）
├── architecture/
│   ├── system.md
│   ├── backend.md
│   ├── frontend.md
│   └── database.md
├── requirements/            # RQ-XXX.md 需求文件
├── tasks/
│   ├── todo/                # 等待分配
│   ├── coding/              # 开发中
│   ├── review/              # 待审查
│   ├── blocked/             # 被阻塞
│   └── done/                # 已完成
└── reviews/                 # Review 结果存档
```

---

## 运行模式

**手动模式**（适合任何 AI 对话工具）

由你驱动每一步：手动开启 Agent、手动移动文件。适合需要审查每步结果的场景。

**自动模式**（需要支持多步骤操作的工具，如 Claude Code、Cursor Agent Mode）

启动一次 Orchestrator，它自动循环决策并驱动 Planner / Coder / Reviewer，直到所有需求完成。

两种模式使用同一套文件结构，可以随时切换。

---

## 快速开始

**第一步：启动 Analyst（推荐）**

新开对话 → 粘贴 `docs/prompt/analyst-prompt.md` → 描述你的项目想法

Analyst 会通过对话收集需求，自动生成 `docs/requirements.md` 和 `docs/requirements/RQ-XXX.md`。

也可以手动填写：
- `docs/requirements.md` — 项目要做什么、有哪些模块
- `docs/architecture/system.md` — 使用什么技术栈
- `docs/conventions.md` — 命名规范和代码风格

**第二步：启动 Planner**

新开对话 → 粘贴 `docs/planner-prompt.md` 中的提示词 → 附上 `docs/` 下所有文档内容

Planner 会在 `tasks/todo/` 下生成若干 TASK 文件。

**第三步：启动 Coder**

新开对话 → 粘贴 `docs/coder-prompt.md` 中的提示词 → 附上 `architecture/*` + `conventions.md` + 当前 TASK 文件

Coder 实现代码后，将 Task 文件手动移到 `tasks/review/`。

**第四步：启动 Reviewer**

新开对话 → 粘贴 `docs/reviewer-prompt.md` 中的提示词 → 附上 TASK 文件 + `architecture/*` + 代码变更内容

根据结果操作：

| 结果 | 操作 |
|------|------|
| PASS | Task 移到 `tasks/done/` |
| FAIL（FailCount < 3）| FailCount+1，Task 移回 `tasks/coding/` |
| FAIL（FailCount ≥ 3）| Task 移到 `tasks/blocked/`，重启 Planner 处理 |
| BLOCKED | Task 移到 `tasks/blocked/`，重启 Planner 处理 |

重复第三、四步直到所有需求完成。

---

## 流程说明

```
需求文档（通过 Analyst 收集或手动填写）
   ↓
[Planner] 拆解任务 → tasks/todo/
   ↓
[Planner] 分配任务 → tasks/coding/
   ↓
[Coder]   实现代码 → tasks/review/
   ↓
[Reviewer] 审查
   ↓
PASS ──────────────────→ tasks/done/
FAIL (FailCount < 3) → tasks/coding/  ← Coder 修复
FAIL (FailCount ≥ 3) → tasks/blocked/
BLOCKED ─────────────→ tasks/blocked/
                              ↓
                        [Planner] 修改需求/架构
                              ↓
                         tasks/todo/ （FailCount 清零）
```

**核心原则：每个 Agent 都新开对话，执行前 `/clear`，不依赖聊天历史。**
