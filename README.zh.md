**中文** | [English](README.md)

# Vibe-Coding 工作流

> Markdown 是记忆，Git 是数据库，飞书多维表格是状态机，Agent 是执行者。

四个 Agent 协作完成开发任务，项目总目标/架构/规范由 Markdown 维护，逐条需求和任务状态由飞书多维表格维护，均不依赖对话历史。

---

## 目录结构

```
docs/
├── requirements.md          # 项目总目标
├── conventions.md           # 编码规范
├── decisions.md             # 架构决策记录
├── process.md               # 完整工作流说明
├── lark-base.md             # 飞书多维表格配置（app_token、table_id、命令参考）
├── prompt/
│   ├── analyst-prompt.md        # Analyst 提示词（需求收集）
│   ├── planner-prompt.md        # Planner 提示词
│   ├── coder-prompt.md          # Coder 提示词
│   ├── reviewer-prompt.md       # Reviewer 提示词
│   └── orchestrator-prompt.md   # Orchestrator 提示词（可选，自动驱动全流程）
└── architecture/
    ├── system.md
    ├── backend.md
    ├── frontend.md
    └── database.md
```

逐条需求（RQ）、任务（Task）、以及审查结果都在 **飞书多维表格** 里，没有本地文件：

- **Requirements 表**：每行一个需求，含状态、功能点、验收标准。
- **Tasks 表**：每行一个 Task，通过双向关联指向所属需求；该 Task 最新一轮的 Review 结果也存在同一行。

Agent 通过 lark-cli 读取和更新记录。

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

Analyst 会通过对话收集需求，自动生成 `docs/requirements.md`，并在 Requirements 表为每个模块创建一条 RQ 记录。

也可以手动填写：
- `docs/requirements.md` — 项目要做什么、有哪些模块
- Requirements 表 — 逐条需求（每行一个 RQ）
- `docs/architecture/system.md` — 使用什么技术栈
- `docs/conventions.md` — 命名规范和代码风格

**第二步：启动 Planner**

新开对话 → 粘贴 `docs/prompt/planner-prompt.md` 中的提示词 → 附上 `docs/` 下所有文档内容，并查询 Requirements 表

Planner 会在 Tasks 表创建若干任务记录（Status=todo，Requirement 关联到对应 RQ），Task 的全部信息都存在记录字段里。

**第三步：启动 Coder**

新开对话 → 粘贴 `docs/prompt/coder-prompt.md` 中的提示词 → 附上 `architecture/*` + `conventions.md`，并查询当前 coding 任务记录

Coder 实现代码后，将该任务的 Base 记录更新为 Status=review。

**第四步：启动 Reviewer**

新开对话 → 粘贴 `docs/prompt/reviewer-prompt.md` 中的提示词 → 查询当前 review 任务记录 + 附上 `architecture/*` + 代码变更内容

把 Review 结果写入该任务记录的 Review 字段，并根据结果更新 Status：

| 结果 | 操作 |
|------|------|
| PASS | Status → done |
| FAIL（FailCount < 3）| FailCount+1，Status → coding |
| FAIL（FailCount ≥ 3）| Status → blocked，重启 Planner 处理 |
| BLOCKED | Status → blocked，重启 Planner 处理 |

重复第三、四步直到所有需求完成。

---

## 流程说明

```
需求（通过 Analyst 收集或手动录入 Requirements 表）
   ↓
[Planner] 拆解任务 → Status=todo
   ↓
[Planner] 分配任务 → Status=coding
   ↓
[Coder]   实现代码 → Status=review
   ↓
[Reviewer] 审查
   ↓
PASS ──────────────────→ Status=done
FAIL (FailCount < 3) → Status=coding  ← Coder 修复
FAIL (FailCount ≥ 3) → Status=blocked
BLOCKED ─────────────→ Status=blocked
                              ↓
                        [Planner] 修改需求/架构
                              ↓
                         Status=todo （FailCount 清零）
```

> 状态流转通过 `lark-cli base +record-upsert --record-id ...` 更新飞书多维表格的 Status 字段完成，不再移动文件。

**核心原则：每个 Agent 都新开对话，执行前 `/clear`，不依赖聊天历史。**
