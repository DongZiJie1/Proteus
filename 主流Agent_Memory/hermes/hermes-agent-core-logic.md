# Hermes Agent 核心逻辑

## 1. 总体架构

**主干 3 层 + 1 个自进化闭环：**

```
触发层 → 网关层 → Agent 层 → 记忆/Skill 沉淀 → 反哺下一次触发
                     └────── 每 ~10-15 次 tool call 触发自进化反思
```

- **触发层**：Message / Heartbeat / Cron / Webhook / Hooks 五种信号源
- **网关层**：一进程多渠道适配器 + 路由器 + 安全网关，只做路由不调 LLM
- **Agent 层**：ReAct 循环（Think → Act → Remember），LLM 自主决策何时结束

---

## 2. Agent 循环：ReAct 范式

Agent 不是一次性推理，而是**"思考 → 行动 → 观察 → 再思考"的迭代循环**，决策权在 LLM 手里。

```
收到消息
   ↓
Think（调 LLM 决策）→ 三选一：
   A. 信息不够 → 调工具 X
   B. 需要回忆 → memory_search
   C. 信息够了 → 输出回复，结束
   ↓
Act（执行工具，拿结果）
   ↓
Remember（读写记忆，可选）
   ↓
工具结果回灌上下文 → 回到 Think
```

### Think 层
- 把上下文（System Prompt + 对话历史 + 工具结果）送 LLM，拿下一步决策
- **主模型 + 辅助模型分工**：强模型做推理，弱模型做压缩/摘要/错误归类

### Act 层
- 工具：Shell 执行、浏览器（CDP）、文件操作、消息发送、Subagent Spawn、MCP 集成
- **6 种终端后端**：local / Docker / SSH / Daytona / Singularity / Modal
- **Error Classifier**：预编码常见失败模式，弱模型直接走预设恢复路径（harness engineering）

### Remember 层
- 记忆分 6 层：Bootstrap / Transcript / Context / Retrieval Index / Skill Library / Honcho

---

## 3. 记忆体系

### 静态结构

```
Context Window（每轮拼装）
   = System Prompt (Bootstrap 文件)
   + 对话切片 (从 .jsonl 加载)
   + 检索到的 md 片段
   + Skill 内容 (命中时加载)
   + Honcho 用户画像摘要
   + 当前消息 + 工具结果
```

| 层 | 存储位置 | 谁写入 | 进 Context 方式 |
|---|---|---|---|
| Bootstrap Files | `~/.hermes/workspace/*.md` | 用户/自进化 | 每轮自动注入 |
| memory/ 笔记 | `~/.hermes/memory/*.md` | Agent/用户/自进化 | 按需检索 |
| Skill Library | `skills/<cat>/<name>/SKILL.md` | 预置/用户/自进化 | 两阶段匹配后加载 |
| Session Transcript | `sessions/<id>.jsonl` | 系统自动追加 | 续会话时切片加载 |
| Honcho User Model | 独立服务 | 自进化持续更新 | 会话启动时注入摘要 |
| Retrieval Index | `memory/<agent>.sqlite` | 文件变化自动重建 | **不进 Context，只是索引** |

### 认知心理学视角
- **陈述性记忆**（记得什么）→ Bootstrap + memory/ + Transcript + Index
- **程序性记忆**（会做什么）→ Skill Library（Hermes 独有）
- **用户建模**（我是谁）→ Honcho（Hermes 独有）

### 动态更新策略

**写入：** 自进化循环自动、会话结束归档、文件监听同步
**读取：** 自动注入（Bootstrap/Transcript/Honcho）+ 按需检索（memory/Skill）
**压缩：** 上下文满了 → 划分保留区/压缩区 → 分块总结 → 合并摘要 → 写回 JSONL
**反思：** 自进化循环（Hermes 独有），见下节

---

## 4. 自进化循环（Hermes 核心创新）

> "like back propagation but for prompts instead of model weights"
> 像反向传播，但更新的不是模型权重，而是 prompt / 记忆 / skill

### 完整流程

```
1. 用户下达任务
2. Agent 执行 Think → Act 循环
3. 每 ~10 次 tool call，主动暂停
4. 自问三件事：
   ① 学到的东西值不值得保留？
   ② 失败的步骤要不要记下来避免？
   ③ 成功流程能不能抽象成可复用 skill？
5. 值得保留 → 分三路沉淀：
   ├── 事实性知识 → memory/YYYY-MM-DD.md
   ├── 程序性流程 → skills/xxx/SKILL.md
   └── 用户偏好   → Honcho User Model
6. 下次类似任务 → 命中 skill，跳过重新推理
```

### vs OpenClaw Pre-Compaction Flush

| | OpenClaw | Hermes |
|---|---|---|
| 触发时机 | 仅在压缩前 | 每 ~15 次 tool call |
| 模式 | 事后归档 | 持续反思 |
| 更新范围 | 只写 memory | memory + skill + Honcho 三路 |
| 核心关注 | 别丢重要信息 | 值不值留 + 失败避免 + 流程抽象 |

**一句话：** OpenClaw 事后归档，Hermes 持续反思。

---

## 5. 多 Agent 架构

**递归 Centralized**：父 Agent 通过 `spawn_subagent` 调用子 Agent，子 Agent 可继续 spawn，仅返回最终结果（不传私有推理过程）。ACP 协议双向支持，可与 OpenClaw 互相调度。
