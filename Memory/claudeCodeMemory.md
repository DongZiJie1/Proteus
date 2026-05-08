# Claude Code 记忆系统完整分析

## 概述

Claude Code 拥有一套**持久化、基于文件的记忆系统**，使 Claude 能够在多次对话之间保留上下文、用户偏好和项目信息。该系统存储在 `~/.claude/projects/{project-path}/memory/` 目录下。

记忆文件**不会自动创建**——需要通过显式指令（如 `/memory` 命令或对话中提到"remember"）来创建和维护。

> 注意：记忆系统**不能**替代 Claude Code 的产品文档、特性、架构或命令行使用说明。这些应参考 [Claude Code 文档](https://docs.anthropic.com/en/docs/claude-code) 和 `/help` 命令。

---

## 记忆系统架构

### 1. 持久化状态（Session-Scoped vs Persistent）

Claude Code 有多层持久化机制：

| 层级 | 存储位置 | 生命周期 | 用途 |
|------|----------|----------|------|
| **Tasks** | 内存（会话内） | 当前会话 | 追踪多步骤工作进度 |
| **Plans** | 内存（会话内） | 当前会话 | 实现方案设计与用户对齐 |
| **Memory** | `~/.claude/projects/.../memory/` | 跨会话持久 | 用户偏好、项目上下文、外部资源引用 |
| **Hooks** | `~/.claude/settings.json` | 永久 | 自动化行为（事件驱动） |
| **Scheduled Tasks** | `.claude/scheduled_tasks.json` | 永久 | 定时/周期性远程 Agent |
| **Agent State** | `.claude/agent/` | 永久 | 持久化 Agent 状态 |
| **Reminders** | `.claude/reminders.json` | 永久 | 提醒事项 |
| **Session State** | `.claude/sessions/` | 永久 | 会话存档 |
| **Statsig Config** | `~/.claude/statsig/` | 永久 | 功能开关与配置缓存 |
| **Credential Store** | `~/.claude/credentials.json` | 永久 | OAuth token 等凭证 |
| **Stats** | `~/.claude/statsig/` | 永久 | 使用统计与缓存 |

> **关键原则**：Tasks 和 Plans 仅在当前会话内有效——跨会话的持久信息应存入 Memory。如果用户要求保存 PR 列表或活动摘要，应询问其中**哪些信息是令人惊讶或非显而易见的**——那才是值得保存的部分。

### 2. CLAUDE.md — 项目级记忆

**位置**：项目根目录、子目录、或 `~/.claude/CLAUDE.md`

**作用**：作为 Claude Code 的"系统提示词"，定义项目规范、架构、常用命令、开发约束等。

**层级优先级**（从高到低）：
1. 企业策略（`~/.claude/CLAUDE.md`）
2. 项目仓库根目录 `CLAUDE.md`（提交到代码库）
3. 项目仓库子目录 `CLAUDE.md`
4. 父目录的 `CLAUDE.md`（向上继承）
5. 当前工作目录的 `.claude/CLAUDE.md`（本地，不提交）

**创建方式**：
- 运行 `/init` 命令自动分析代码库并生成初始 `CLAUDE.md`
- 手动创建和编辑

**适用场景**：
- 构建/测试/lint 命令（如 `npm run test:unit`）
- 代码风格偏好（如"用 tabs 而非 spaces"）
- 项目架构概述（如"使用 monorepo，app 在 `/packages/app`"）
- 重要注意事项（如"修改 `/generated/schema.ts` 后需重新运行 codegen"）

### 3. Auto Memory — 自动记忆系统

**存储路径**：`~/.claude/projects/{project-path}/memory/`

**索引文件**：`MEMORY.md`（始终加载到对话上下文中，200 行后截断）

**记忆文件格式**（Markdown + YAML frontmatter）：

```markdown
---
name: {{记忆名称}}
description: {{一行描述——用于判断未来对话的相关性，需具体}}
type: {{user, feedback, project, reference}}
---

{{记忆内容——feedback/project 类型结构为：规则/事实，然后是 **Why:** 和 **How to apply:** 行}}
```

---

## 四种记忆类型

### 1. `user` — 用户画像

**触发时机**：了解到用户的角色、偏好、职责或知识背景时

**用途**：帮助 Claude 根据用户的视角和偏好调整行为。例如，资深工程师和编程初学者需要不同的协作方式。

**示例**：
- 用户："I'm a data scientist investigating what logging we have in place"
- 保存：`user is a data scientist, currently focused on observability/logging`

- 用户："I've been writing Go for ten years but this is my first time touching the React side of this repo"
- 保存：`deep Go expertise, new to React and this project's frontend — frame frontend explanations in terms of backend analogues`

### 2. `feedback` — 行为反馈

**触发时机**：
- 用户纠正做法时（"no not that", "don't", "stop doing X"）
- 用户确认非显而易见的做法有效时（"yes exactly", "perfect, keep doing that"）

**结构**：规则本身 → **Why:**（用户给出的原因）→ **How to apply:**（何时/何处适用）

**重要**：不仅要记录纠正，也要记录确认——否则会变得过于保守。

**示例**：
- 用户："don't mock the database in these tests — we got burned last quarter when mocked tests passed but the prod migration failed"
- 保存：`integration tests must hit a real database, not mocks. Reason: prior incident where mock/prod divergence masked a broken migration`

- 用户："yeah the single bundled PR was the right call here, splitting this one would've just been churn"
- 保存：`for refactors in this area, user prefers one bundled PR over many small ones. Confirmed after I chose this approach — a validated judgment call, not a correction`

### 3. `project` — 项目上下文

**触发时机**：了解到项目中的人员、目标、计划、bug 或事件时

**注意**：这些状态变化较快，需保持更新。始终将相对日期转为绝对日期（如"Thursday" → "2026-03-05"）。

**示例**：
- 用户："we're freezing all non-critical merges after Thursday — mobile team is cutting a release branch"
- 保存：`merge freeze begins 2026-03-05 for mobile release cut. Flag any non-critical PR work scheduled after that date`

- 用户："the reason we're ripping out the old auth middleware is that legal flagged it for storing session tokens in a way that doesn't meet the new compliance requirements"
- 保存：`auth middleware rewrite is driven by legal/compliance requirements around session token storage, not tech-debt cleanup — scope decisions should favor compliance over ergonomics`

### 4. `reference` — 外部资源引用

**触发时机**：了解到外部系统中的资源及其用途时

**示例**：
- 用户："check the Linear project 'INGEST' if you want context on these tickets, that's where we track all pipeline bugs"
- 保存：`pipeline bugs are tracked in Linear project "INGEST"`

- 用户："the Grafana board at grafana.internal/d/api-latency is what oncall watches"
- 保存：`grafana.internal/d/api-latency is the oncall latency dashboard — check it when editing request-path code`

---

## 不应保存到 Memory 的内容

| 不应保存 | 原因 |
|----------|------|
| 代码模式、约定、架构、文件路径、项目结构 | 可从当前项目状态推导 |
| Git 历史、最近变更、谁改了什么 | `git log` / `git blame` 是权威来源 |
| 调试方案或修复方法 | 修复在代码中，commit message 有上下文 |
| CLAUDE.md 中已有的内容 | 重复 |
| 临时任务详情（进行中的工作、临时状态、当前对话上下文） | 过时快 |

---

## Memory 操作指南

### 保存记忆（两步流程）

**Step 1** — 将记忆写入独立文件（如 `user_role.md`）：

```markdown
---
name: {{memory name}}
description: {{一行描述}}
type: {{user, feedback, project, reference}}
---

{{记忆内容}}
```

**Step 2** — 在 `MEMORY.md` 中添加指向该文件的索引：

```markdown
- [Title](file.md) — 一行摘要
```

### MEMORY.md 规则

- 始终加载到对话上下文中
- **200 行后截断**，需保持索引简洁
- 每条记录不超过 ~150 字符
- 按语义主题组织，不按时间顺序
- 更新或删除过时/错误的记忆
- **不写重复记忆**——先检查是否有可更新的现有记忆

### 访问记忆的时机

- 记忆似乎相关时
- 用户引用先前对话工作时
- **用户明确要求检查/回忆/记住时，必须访问**

### 使用记忆前的验证

> "记忆说 X 存在" ≠ "X 现在存在"

- 记忆提到文件路径 → 检查文件是否存在
- 记忆提到函数或标志 → grep 搜索确认
- 记忆总结仓库状态（活动日志、架构快照）→ 冻结在写入时间点；询问*最近/当前*状态时，优先用 `git log` 或读代码

---

## 斜杠命令

### `/memory` — 原子化记忆编辑

- 打开一个原子化的批量编辑流程
- 用于在单次操作中创建、修改或删除多条记忆
- 适合大型重构或定期维护
- 不能用于调度操作

### `/remember` — 快速记忆保存

- 在活跃编码会话中快速保存单条记忆
- 捕获事实、偏好或反馈以供未来对话使用
- 适合在工作中快速记录

---

## 记忆与其他持久化机制的对比

| 机制 | 何时使用 |
|------|----------|
| **Memory** | 未来对话中有用的信息（跨会话） |
| **Plan** | 当前任务的实现方案（当前会话内，需与用户对齐） |
| **Tasks** | 当前对话的多步骤工作追踪（当前会话内） |

**判断逻辑**：
- 如果信息仅在当前对话有用 → 不存 Memory
- 如果需要跨会话保留 → 存 Memory
- 如果是实现方案设计 → 用 Plan
- 如果是任务进度追踪 → 用 Tasks

---

## Hooks（钩子）

**配置位置**：`~/.claude/settings.json`

**作用**：实现自动化行为（"当 X 发生时，自动执行 Y"）。Claude 无法直接设置 hook，必须通过 Settings 命令或手动编辑 `settings.json`。

**用途**：
- 自动化重复性工作流
- 强制编码标准
- 连接外部系统
- 创建事件驱动的自动化

---

## Scheduled Tasks（定时任务）

**存储位置**：`.claude/scheduled_tasks.json`

**作用**：运行重复性的 remote Agent——自主的 Claude Code 实例，可独立访问你的代码库。

**特性**：
- 可通过 UI、`/schedule` 命令或 API 配置
- 支持 cron 表达式定义执行频率
- Agent 输出可通过 Webhook URL 推送到 Slack 等平台
- 可与 GitHub PR 集成
- 会话级定时任务 7 天后自动过期

---

## 对 Proteus 项目的启示

作为类似 Claude Code 的 AI Agent 工具，Proteus 可以借鉴的记忆系统设计：

1. **分层记忆架构**：CLAUDE.md（项目级） + Memory（用户级/跨会话） + Tasks/Plans（会话级）
2. **类型化记忆**：user / feedback / project / reference 四种类型，各有不同的触发条件和使用场景
3. **索引文件**：`MEMORY.md` 作为轻量索引，避免上下文窗口被大量记忆内容淹没
4. **验证机制**：使用记忆前验证其引用的内容是否仍然存在
5. **边界清晰**：明确什么该存、什么不该存，避免记忆系统变成垃圾堆积场
6. **与任务系统解耦**：Memory 跨会话，Tasks/Plans 仅会话内，各司其职
