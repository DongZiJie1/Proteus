# s11: Autonomous Agents (自治智能体)

`s01 > s02 > s03 > s04 > s05 > s06 | s07 > s08 > s09 > s10 > [ s11 ] s12`

> *"队友自己看看板, 有活就认领"* -- 不需要领导逐个分配, 自组织。

## 问题

s09-s10 中, 队友只在被明确指派时才动。领导得给每个队友写 prompt, 任务看板上 10 个未认领的任务得手动分配。这扩展不了。

真正的自治: 队友自己扫描任务看板, 认领没人做的任务, 做完再找下一个。

一个细节: 上下文压缩 (s06) 后智能体可能忘了自己是谁。身份重注入解决这个问题。

## 解决方案

```
Teammate lifecycle with idle cycle:

+-------+
| spawn |
+---+---+
    |
    v
+-------+   tool_use     +-------+
| WORK  | <------------- |  LLM  |
+---+---+                +-------+
    |
    | stop_reason != tool_use (or idle tool called)
    v
+--------+
|  IDLE  |  poll every 5s for up to 60s
+---+----+
    |
    +---> check inbox --> message? ----------> WORK
    |
    +---> scan .tasks/ --> unclaimed? -------> claim -> WORK
    |
    +---> 60s timeout ----------------------> SHUTDOWN

Identity re-injection after compression:
  if len(messages) <= 3:
    messages.insert(0, identity_block)
```

## 工作原理

1. 队友循环分两个阶段: WORK 和 IDLE。LLM 停止调用工具 (或调用了 `idle`) 时, 进入 IDLE。

```python
def _loop(self, name, role, prompt):
    while True:
        # -- WORK PHASE --
        messages = [{"role": "user", "content": prompt}]
        for _ in range(50):
            response = client.messages.create(...)
            if response.stop_reason != "tool_use":
                break
            # execute tools...
            if idle_requested:
                break

        # -- IDLE PHASE --
        self._set_status(name, "idle")
        resume = self._idle_poll(name, messages)
        if not resume:
            self._set_status(name, "shutdown")
            return
        self._set_status(name, "working")
```

2. 空闲阶段循环轮询收件箱和任务看板。

```python
def _idle_poll(self, name, messages):
    for _ in range(IDLE_TIMEOUT // POLL_INTERVAL):  # 60s / 5s = 12
        time.sleep(POLL_INTERVAL)
        inbox = BUS.read_inbox(name)
        if inbox:
            messages.append({"role": "user",
                "content": f"<inbox>{inbox}</inbox>"})
            return True
        unclaimed = scan_unclaimed_tasks()
        if unclaimed:
            claim_task(unclaimed[0]["id"], name)
            messages.append({"role": "user",
                "content": f"<auto-claimed>Task #{unclaimed[0]['id']}: "
                           f"{unclaimed[0]['subject']}</auto-claimed>"})
            return True
    return False  # timeout -> shutdown
```

3. 任务看板扫描: 找 pending 状态、无 owner、未被阻塞的任务。

```python
def scan_unclaimed_tasks() -> list:
    unclaimed = []
    for f in sorted(TASKS_DIR.glob("task_*.json")):
        task = json.loads(f.read_text())
        if (task.get("status") == "pending"
                and not task.get("owner")
                and not task.get("blockedBy")):
            unclaimed.append(task)
    return unclaimed
```

4. 身份重注入: 上下文过短 (说明发生了压缩) 时, 在开头插入身份块。

```python
if len(messages) <= 3:
    messages.insert(0, {"role": "user",
        "content": f"<identity>You are '{name}', role: {role}, "
                   f"team: {team_name}. Continue your work.</identity>"})
    messages.insert(1, {"role": "assistant",
        "content": f"I am {name}. Continuing."})
```

## claim_task 的实现

文档展示了 `scan_unclaimed_tasks`，但 `claim_task` 同样重要 -- 它负责写入 owner 字段，把任务从"无主"变成"有主"：

```python
def claim_task(task_id: int, owner: str):
    task_path = TASKS_DIR / f"task_{task_id}.json"
    task = json.loads(task_path.read_text())
    task["owner"] = owner
    task["status"] = "in_progress"
    task_path.write_text(json.dumps(task, indent=2))
```

逻辑很直接：读文件、写 owner、推进状态到 `in_progress`、存回去。但这里有一个隐患，下面"设计取舍"里会讲。

## 设计取舍与局限性

**任务认领的竞态条件**: 这是 s11 最大的隐患。两个队友同时调用 `scan_unclaimed_tasks()`，都看到 task_3 没人认领，都去 `claim_task(3, ...)`，后写入的覆盖先写入的 -- 两个队友都以为自己认领成功了，实际上只有一个的 owner 留了下来，另一个在白干。

```
时间线:
  alice: scan() -> 看到 task_3 无主 -> claim(3, "alice") -> 写入 owner="alice"
  bob:   scan() -> 看到 task_3 无主 -> claim(3, "bob")   -> 写入 owner="bob" (覆盖!)
  结果: alice 以为自己认领了 task_3，但磁盘上 owner 是 bob
```

生产环境需要原子认领，常见做法：

```python
import fcntl

def claim_task_safe(task_id: int, owner: str) -> bool:
    task_path = TASKS_DIR / f"task_{task_id}.json"
    with open(task_path, "r+") as f:
        fcntl.flock(f, fcntl.LOCK_EX)  # 排他锁
        task = json.loads(f.read())
        if task.get("owner"):  # 已经被别人认领了
            fcntl.flock(f, fcntl.LOCK_UN)
            return False
        task["owner"] = owner
        task["status"] = "in_progress"
        f.seek(0)
        f.write(json.dumps(task, indent=2))
        f.truncate()
        fcntl.flock(f, fcntl.LOCK_UN)
    return True
```

或者更简单的方案：用 `os.rename()` 的原子性，认领时把 `task_3.json` 重命名为 `task_3.alice.json`，重命名失败说明被别人抢了。

**空闲轮询的效率问题**: 每 5 秒轮询一次是 busy-wait 的变体。任务少的时候 12 次轮询（60 秒）大部分是浪费；任务密集时 5 秒的间隔又意味着新任务最多要等 5 秒才被发现。更好的方案是用文件系统事件通知（`inotify` / `kqueue`）或条件变量（`threading.Event`）：

```python
# 用 Event 替代 sleep 轮询
task_event = threading.Event()

# 创建新任务时通知
def create_task(...):
    ...
    task_event.set()  # 唤醒所有等待的队友

# 空闲阶段等待
def _idle_poll(self, name, messages):
    for _ in range(IDLE_TIMEOUT // POLL_INTERVAL):
        task_event.wait(timeout=POLL_INTERVAL)  # 有事件立刻醒，没事件等 5 秒
        task_event.clear()
        ...
```

**60 秒超时的刚性**: 超时是硬编码的。如果一个长任务在第 61 秒才被创建，队友已经 shutdown 了，需要重新 spawn。没有动态调整机制，也没有"快要超时了先别关"的预警。生产环境可以用指数退避或根据任务看板活跃度动态调整超时。

**自动关机绕过了 s10 的关机协议**: s10 精心设计了 shutdown 的请求-响应握手，但 s11 的超时关机直接调 `self._set_status(name, "shutdown")` 就退出了，没有通知领导、没有走协议。领导不知道队友是正常关机还是崩溃了。理想情况下，超时关机应该给领导发一条消息：

```python
if not resume:
    BUS.send(name, "lead", f"{name} shutting down: idle timeout",
             msg_type="shutdown_notice")
    self._set_status(name, "shutdown")
    return
```

**无任务亲和性**: `scan_unclaimed_tasks` 返回的是按文件名排序的第一个可用任务，不考虑队友的角色。一个 role="frontend" 的队友可能认领了 "数据库迁移" 任务。可以加一个简单的匹配：

```python
def scan_unclaimed_tasks(role=None) -> list:
    unclaimed = []
    for f in sorted(TASKS_DIR.glob("task_*.json")):
        task = json.loads(f.read_text())
        if (task.get("status") == "pending"
                and not task.get("owner")
                and not task.get("blockedBy")):
            # 如果任务有 preferred_role 且不匹配，跳过
            if role and task.get("preferred_role") and task["preferred_role"] != role:
                continue
            unclaimed.append(task)
    return unclaimed
```

## 补充理解

### 为什么需要显式的 idle 工具？

s09 的队友循环靠 `stop_reason != "tool_use"` 判断"活干完了"。但 LLM 停止调用工具有两种情况：

1. 真的没活干了，想进入空闲
2. 只是想说句话（比如 "我完成了 task_3，结果如下..."），说完还要继续

没有 idle 工具时，这两种情况无法区分。LLM 说了句话就被踢进 IDLE，可能它其实还想继续工作。

`idle` 工具是一个明确的信号："我确认没有更多工作要做了，请让我进入空闲状态"。这把"是否空闲"的决策权交给了 LLM 自己，而不是靠外部推断。

```python
# idle 工具的定义
{
    "name": "idle",
    "description": "Call this when you have no more work to do. "
                   "You will enter idle mode and poll for new tasks.",
    "input_schema": {"type": "object", "properties": {}}
}
```

### 为什么身份重注入的阈值是 len(messages) <= 3？

回顾 s06 的 auto_compact：压缩后 messages 被替换为两条消息：

```python
# auto_compact 的输出
[
    {"role": "user", "content": "[Compressed]\n\n{summary}"},      # 第 1 条
    {"role": "assistant", "content": "Understood. Continuing."},   # 第 2 条
]
```

然后 IDLE 阶段唤醒时会追加一条新消息（收件箱内容或认领的任务）：

```python
messages.append({"role": "user", "content": f"<auto-claimed>..."})  # 第 3 条
```

所以 `len(messages) <= 3` 的含义是："如果消息列表只有压缩摘要 + 一条新任务，说明刚经历过压缩，身份信息可能丢了"。

如果没有发生过压缩，messages 里会有完整的对话历史，远超 3 条，身份信息还在上下文里，不需要重注入。

### 循环结构的变化：为什么 s11 重构了 agent loop？

对比 s09 的循环：

```python
# s09: 单阶段，干完就结束
def _teammate_loop(self, name, role, prompt):
    messages = [{"role": "user", "content": prompt}]
    for _ in range(50):
        # ... LLM 调用 + 工具执行 ...
        if response.stop_reason != "tool_use":
            break
    self._find_member(name)["status"] = "idle"
    # 结束，线程退出
```

s11 在外面套了一层 `while True`：

```python
# s11: 双阶段，干完进入空闲，有活再干
def _loop(self, name, role, prompt):
    while True:
        # WORK: 内层循环（和 s09 一样）
        for _ in range(50):
            ...
        # IDLE: 轮询等待
        resume = self._idle_poll(name, messages)
        if not resume:
            return  # 真正退出
        # 有活了，回到 WORK
```

s09 的队友是"一次性工人"：领导给一个任务，干完就退休。s11 的队友是"长期员工"：干完一个任务不走，等着看有没有下一个。外层 `while True` 就是这个"不走"的实现。

### 自组织 vs 领导指派：什么时候该用哪个？

自组织不是银弹。两种模式各有适用场景：

| 维度 | 领导指派 (s10) | 自组织 (s11) |
|------|---------------|-------------|
| 适合 | 任务需要特定技能、有复杂依赖、需要全局视角 | 任务同质化、互相独立、数量多 |
| 类比 | 手术团队：主刀医生指挥每个人做什么 | 快递分拣：谁空了谁拿下一个包裹 |
| 风险 | 领导成为瓶颈 | 任务分配不合理（前端干了后端的活） |
| 扩展性 | 差（领导带宽有限） | 好（加人就能加速） |

实际系统中通常混合使用：领导负责拆解目标、创建任务、设置依赖和 preferred_role，队友自己去认领匹配自己角色的任务。领导从"逐个分配"变成"设计任务看板"。

### 与 s12 的衔接：自治 + 隔离

s11 解决了"谁来做"的问题，但所有队友还是在同一个目录里干活。alice 改 `config.py`，bob 也改 `config.py`，互相踩脚。s12 给每个任务分配独立的 git worktree，解决"在哪做"的问题。

```
s11 单独:  alice claim task_1 -> 改 config.py
           bob   claim task_2 -> 也改 config.py  -> 冲突!

s11 + s12: alice claim task_1 -> .worktrees/auth-refactor/config.py
           bob   claim task_2 -> .worktrees/ui-login/config.py     -> 隔离!
```

s11 的 `claim_task` 在 s12 中会扩展为 `claim_task` + `create_worktree`，认领任务的同时创建隔离目录。

### 与真实产品的对比

| 特性 | 本教程 (s11) | Devin | GitHub Copilot Workspace | Claude Code |
|------|-------------|-------|--------------------------|-------------|
| 自治认领 | 轮询任务看板 | 自主规划 + 执行 | 用户驱动，非自治 | 无（单 agent） |
| 空闲机制 | 60s 轮询后关机 | 持续运行 | 无（按需启动） | 无 |
| 身份管理 | 压缩后重注入 | 持久身份 | 无需（无压缩） | system prompt |
| 多 agent | 线程级并发 | 单 agent + 工具 | 单 agent | 单 agent |
| 任务分配 | 自动 + 无亲和性 | 自主决策 | 人工选择 | 人工指令 |

目前大多数产品还是单 agent 架构。多 agent 自组织更多出现在研究原型和框架（如 AutoGen、CrewAI）中。s11 的设计思路和 CrewAI 的 "autonomous task delegation" 比较接近，但实现更轻量。

## 相对 s10 的变更

| 组件           | 之前 (s10)       | 之后 (s11)                       |
|----------------|------------------|----------------------------------|
| Tools          | 12               | 14 (+idle, +claim_task)          |
| 自治性         | 领导指派         | 自组织                           |
| 空闲阶段       | 无               | 轮询收件箱 + 任务看板            |
| 任务认领       | 仅手动           | 自动认领未分配任务               |
| 身份           | 系统提示         | + 压缩后重注入                   |
| 超时           | 无               | 60 秒空闲 -> 自动关机            |

## 试一试

```sh
cd learn-claude-code
python agents/s11_autonomous_agents.py
```

试试这些 prompt (英文 prompt 对 LLM 效果更好, 也可以用中文):

1. `Create 3 tasks on the board, then spawn alice and bob. Watch them auto-claim.`
2. `Spawn a coder teammate and let it find work from the task board itself`
3. `Create tasks with dependencies. Watch teammates respect the blocked order.`
4. 输入 `/tasks` 查看带 owner 的任务看板
5. 输入 `/team` 监控谁在工作、谁在空闲
