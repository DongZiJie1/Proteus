# Claude Code 智能体架构精华总结

> 从零构建一个完整的多智能体编程助手，12 步渐进式演进。

## 核心架构：一个循环统治一切

```python
def agent_loop(query):
    messages = [{"role": "user", "content": query}]
    while True:
        response = client.messages.create(model=MODEL, messages=messages, tools=TOOLS)
        messages.append({"role": "assistant", "content": response.content})
        if response.stop_reason != "tool_use":
            return  # 模型不再调用工具，结束
        for block in response.content:
            if block.type == "tool_use":
                output = TOOL_HANDLERS[block.name](**block.input)
                messages.append({"role": "user", "content": [{"type": "tool_result", "tool_use_id": block.id, "content": output}]})
```

**这 20 行代码是整个系统的骨架，后续 11 章都在这个循环上叠加机制，循环本身始终不变。**

---

## 演进路线图

```
基础能力 (s01-s06)                    协作能力 (s07-s12)
┌─────────────────────────┐          ┌─────────────────────────┐
│ s01 Agent Loop          │          │ s07 Task System         │
│   └─ 一个循环 + bash     │          │   └─ 持久化任务图 (DAG)  │
│ s02 Tool Use            │          │ s08 Background Tasks    │
│   └─ dispatch map 扩展   │          │   └─ 后台线程 + 通知队列 │
│ s03 TodoWrite           │          │ s09 Agent Teams         │
│   └─ 内存计划 + nag      │          │   └─ 队友 + JSONL 邮箱   │
│ s04 Subagent            │          │ s10 Team Protocols      │
│   └─ 隔离上下文          │          │   └─ request-response   │
│ s05 Skill Loading       │          │ s11 Autonomous Agents   │
│   └─ 按需加载知识        │          │   └─ 自组织认领任务      │
│ s06 Context Compact     │          │ s12 Worktree Isolation  │
│   └─ 三层压缩策略        │          │   └─ git worktree 隔离  │
└─────────────────────────┘          └─────────────────────────┘
```

---

## 六大核心机制

### 1. 工具扩展：Dispatch Map 模式

```python
TOOL_HANDLERS = {
    "bash":       lambda **kw: run_bash(kw["command"]),
    "read_file":  lambda **kw: run_read(kw["path"]),
    "write_file": lambda **kw: run_write(kw["path"], kw["content"]),
}
# 加工具 = 加 handler + 加 schema，循环永远不变
```

**关键洞察**：所有工具通过字典查找分发，新增工具不需要改循环代码。

### 2. 计划追踪：TodoManager + Nag Reminder

```python
# 模型连续 3 轮不更新计划时，注入提醒
if rounds_since_todo >= 3:
    messages[-1]["content"].insert(0, {"type": "text", "text": "<reminder>Update your todos.</reminder>"})
```

**关键洞察**：强制"同时只能有一个 in_progress"，制造聚焦压力；nag reminder 制造问责压力。

### 3. 上下文管理：三层压缩策略

| 层级 | 触发条件 | 操作 | 代价 |
|------|----------|------|------|
| micro_compact | 每轮 | 旧 tool_result → 占位符 | 几乎为零 |
| auto_compact | token > 50k | LLM 摘要 + 存 transcript | 一次 LLM 调用 |
| manual compact | 显式调用 | 同 auto_compact | 一次 LLM 调用 |

**关键洞察**：能省则省，能晚则晚。micro_compact 处理 80% 的膨胀问题。

### 4. 子智能体：隔离上下文

```python
def run_subagent(prompt: str) -> str:
     = [{"role": "user", "content": prompt}]  # 全新上下文
    # ... 运行完整 agent loop ...
    return final_text  # 只返回摘要，丢弃整个 messages
```

**关键洞察**：子智能体 = 一次性上下文沙箱。30 次工具调用的历史直接丢弃，父智能体只收到一句摘要。

### 5. 技能加载：两层注入

```
第一层 (system prompt): 技能名 + 描述 (~100 tokens/skill)
第二层 (tool_result):   完整内容 (~2000 tokens)，按需加载
```

**关键洞察**：模型知道有什么技能（便宜），需要时再加载完整内容（贵）。技能可丢弃、可重载，天然适配上下文压缩。

### 6. 任务图：DAG + 磁盘持久化

```
.tasks/
  task_1.json  {"id":1, "status":"completed", "blockedBy":[], "blocks":[2,3]}
  task_2.json  {"id":2, "status":"pending", "blockedBy":[1]}
  task_3.json  {"id":3, "status":"pending", "blockedBy":[1]}
```

**关键洞察**：
- 每任务一个文件 → 并发安全，原子写入
- `blockedBy` 是数组 → 支持多依赖（DAG 而非树）
- 磁盘持久化 → 上下文压缩后不丢失，崩溃可恢复

---

## 多智能体协作架构

### 通信：JSONL 邮箱

```python
# 发送：追加一行
with open(f"{to}.jsonl", "a") as f:
    f.write(json.dumps({"from": sender, "content": content}) + "\n")

# 接收：读取全部 + 清空
msgs = [json.loads(l) for l in path.read_text().splitlines()]
path.write_text("")  # drain
```

### 协议：Request-Response 模式

```
请求: {req_id: "abc", action: "shutdown", target: "alice"}
响应: {req_id: "abc", approve: true}
         ^^^
         同一个 ID，关联请求和响应
```

**关键洞察**：一个 FSM (`pending → approved | rejected`)，两种用途（关机协议、计划审批）。协议不需要改底层基础设施，只需要在现有消息通道上定义新的消息格式。

### 自组织：空闲轮询 + 任务认领

```python
def _idle_poll(self, name):
    for _ in range(12):  # 60s / 5s
        time.sleep(5)
        if inbox := BUS.read_inbox(name):
            return True  # 有消息，继续工作
        if unclaimed := scan_unclaimed_tasks():
            claim_task(unclaimed[0]["id"], name)
            return True  # 认领任务，继续工作
    return False  # 超时，关机
```

**关键洞察**：队友自己扫描任务看板，认领没人做的任务，做完再找下一个。领导从"逐个分配"变成"设计任务看板"。

### 隔离：Git Worktree

```
主仓库/              ← main 分支
.worktrees/
  auth-refactor/    ← wt/auth-refactor 分支
  ui-login/         ← wt/ui-login 分支
```

**关键洞察**：
- 文件系统层隔离：不同目录，`cwd` 不同
- Git 层隔离：不同分支，未提交改动互不可见
- 任务完成后正常 merge 回主分支

---

## 状态管理哲学

| 状态类型 | 存储位置 | 生命周期 | 示例 |
|----------|----------|----------|------|
| 会话状态 | messages[] | 易失 | 对话历史 |
| 计划状态 | 内存 (TodoManager) | 会话内 | 当前步骤 |
| 任务状态 | 磁盘 (.tasks/) | 持久 | 任务图 |
| 通信状态 | 磁盘 (JSONL) | 持久 | 收件箱 |
| 执行状态 | 磁盘 (worktree) | 持久 | 代码改动 |

**核心原则**：会话记忆是易失的，磁盘状态是持久的。崩溃后从磁盘重建现场。

---

## 关键设计取舍

| 问题 | 教学实现 | 生产环境应该 |
|------|----------|--------------|
| 任务认领竞态 | 无锁，可能重复认领 | 文件锁或原子重命名 |
| 空闲轮询效率 | 固定 5s 间隔 | inotify/Event 通知 |
| 关机协议 | 超时直接退出 | 通知领导 + 优雅关闭 |
| 消息确认 | 无 ACK | 确认 + 重试机制 |
| 环检测 | 无 | 拓扑排序检测 |

---

## 一句话总结每一步

| 步骤 | 核心问题 | 解决方案 |
|------|----------|----------|
| s01 | 模型碰不到真实世界 | while 循环 + bash 工具 |
| s02 | 只有 bash 不够安全 | dispatch map + 路径沙箱 |
| s03 | 多步任务会跑偏 | TodoManager + nag reminder |
| s04 | 上下文越来越胖 | 子智能体隔离上下文 |
| s05 | 知识全塞 system prompt 太浪费 | 两层注入，按需加载 |
| s06 | 上下文窗口会满 | 三层压缩 + transcript 备份 |
| s07 | 扁平清单没有依赖关系 | DAG 任务图 + 磁盘持久化 |
| s08 | 慢命令阻塞循环 | 后台线程 + 通知队列 |
| s09 | 子智能体是一次性的 | 持久队友 + JSONL 邮箱 |
| s10 | 队友之间没有协调规矩 | request-response 协议 |
| s11 | 领导要逐个分配任务 | 自组织认领 + 空闲轮询 |
| s12 | 多智能体共享目录会冲突 | git worktree 隔离 |

---

## 架构全景图

```
┌─────────────────────────────────────────────────────────────────┐
│                         User Interface                          │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Lead Agent (主循环)                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │
│  │ TodoManager │  │ SkillLoader │  │ Compactor   │              │
│  │   (s03)     │  │   (s05)     │  │   (s06)     │              │
│  └─────────────┘  └─────────────┘  └─────────────┘              │
└─────────────────────────────────────────────────────────────────┘
         │                    │                    │
         ▼                    ▼                    ▼
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│  Tool Dispatch  │  │  Subagent Pool  │  │ Background Mgr  │
│     (s02)       │  │     (s04)       │  │     (s08)       │
└─────────────────┘  └─────────────────┘  └─────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Teammate Manager (s09)                      │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │
│  │   Alice     │  │    Bob      │  │   Charlie   │              │
│  │  (thread)   │  │  (thread)   │  │  (thread)   │              │
│  └─────────────┘  └─────────────┘  └─────────────┘              │
└─────────────────────────────────────────────────────────────────┘
         │                    │                    │
         ▼                    ▼                    ▼
┌─────────────────────────────────────────────────────────────────┐
│                       Message Bus (JSONL)                        │
│            inbox/alice.jsonl  inbox/bob.jsonl  ...              │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Persistent State Layer                        │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │
│  │ .tasks/     │  │ .worktrees/ │  │.transcripts/│              │
│  │ (DAG)       │  │ (isolation) │  │ (recovery)  │              │
│  └─────────────┘  └─────────────┘  └─────────────┘              │
└─────────────────────────────────────────────────────────────────┘
```

---

## 核心洞察

1. **循环不变原则**：所有扩展都是在同一个 agent loop 上叠加机制，循环本身从 s01 到 s12 一行没改。

2. **磁盘即真相**：会话记忆是易失的，磁盘状态是持久的。任务图、收件箱、worktree 索引都在磁盘上，崩溃后可恢复。

3. **按需加载**：技能、上下文、子智能体都是"用到才加载，用完就丢弃"的模式，最大化利用有限的上下文窗口。

4. **协议不改基础设施**：shutdown 协议、plan approval 协议都跑在 s09 的消息总线上，总线本身一行代码没动。

5. **隔离是并发的前提**：子智能体隔离上下文，worktree 隔离文件系统，才能让多个智能体真正并行工作。
