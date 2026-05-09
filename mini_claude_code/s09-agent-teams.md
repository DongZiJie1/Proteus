# s09: Agent Teams (智能体团队)

`s01 > s02 > s03 > s04 > s05 > s06 | s07 > s08 > [ s09 ] s10 > s11 > s12`

> *"任务太大一个人干不完, 要能分给队友"* -- 持久化队友 + JSONL 邮箱。

## 问题

子智能体 (s04) 是一次性的: 生成、干活、返回摘要、消亡。没有身份, 没有跨调用的记忆。后台任务 (s08) 能跑 shell 命令, 但做不了 LLM 引导的决策。

真正的团队协作需要三样东西: (1) 能跨多轮对话存活的持久智能体, (2) 身份和生命周期管理, (3) 智能体之间的通信通道。

## 解决方案

```
Teammate lifecycle:
  spawn -> WORKING -> IDLE -> WORKING -> ... -> SHUTDOWN

Communication:
  .team/
    config.json           <- team roster + statuses
    inbox/
      alice.jsonl         <- append-only, drain-on-read
      bob.jsonl
      lead.jsonl

              +--------+    send("alice","bob","...")    +--------+
              | alice  | -----------------------------> |  bob   |
              | loop   |    bob.jsonl << {json_line}    |  loop  |
              +--------+                                +--------+
                   ^                                         |
                   |        BUS.read_inbox("alice")          |
                   +---- alice.jsonl -> read + drain ---------+
```

## 工作原理

1. TeammateManager 通过 config.json 维护团队名册。

```python
class TeammateManager:
    def __init__(self, team_dir: Path):
        self.dir = team_dir
        self.dir.mkdir(exist_ok=True)
        self.config_path = self.dir / "config.json"
        self.config = self._load_config()
        self.threads = {}
```

2. `spawn()` 创建队友并在线程中启动 agent loop。

```python
def spawn(self, name: str, role: str, prompt: str) -> str:
    member = {"name": name, "role": role, "status": "working"}
    self.config["members"].append(member)
    self._save_config()
    thread = threading.Thread(
        target=self._teammate_loop,
        args=(name, role, prompt), daemon=True)
    thread.start()
    return f"Spawned teammate '{name}' (role: {role})"
```

> **为什么用线程? 为什么是守护线程 (daemon=True)?**
>
> **为什么需要线程**: 主程序本身就是 lead agent 的交互循环 (REPL), 它要持续接收用户输入、协调任务、收发消息。如果在主程序里直接调 `_teammate_loop("alice", ...)`, 主程序就卡住了, 要等 alice 跑完才能继续。没法同时 spawn bob, 也没法处理用户输入, "团队"就变成串行了。线程让每个队友在后台独立运行, lead 可以随时继续接活:
>
> ```
> 用户输入 -> lead 处理 -> spawn 队友 -> 队友后台干活
>    ^                                        |
>    |         用户继续输入新指令               |
>    +---- lead 随时可以 /team 查状态 ---------+
> ```
>
> **为什么是 daemon=True**: 这是教学代码的简化选择。daemon=True 意味着主进程退出时守护线程会被直接杀掉, 省掉了整套优雅关闭的逻辑。但代价是: 如果用户 Ctrl+C 退出, 还在干活的队友会被粗暴终止, 可能丢失正在进行的工作。
>
> **生产环境应该怎么做**: 用普通线程 (daemon=False) + 优雅关闭:
>
> ```python
> # spawn 时不设 daemon
> thread = threading.Thread(target=self._teammate_loop, args=(...))
> thread.start()
>
> # 退出时通过消息通知队友收尾, 再 join 等待
> def shutdown(self):
>     for name in self.threads:
>         BUS.send("lead", name, "shutdown", msg_type="control")
>     for t in self.threads.values():
>         t.join(timeout=30)  # 最多等 30 秒
> ```
>
> 这样队友收到 shutdown 消息后可以保存状态、完成当前步骤, 再自行退出。

3. MessageBus: append-only 的 JSONL 收件箱。`send()` 追加一行; `read_inbox()` 读取全部并清空。

```python
class MessageBus:
    def send(self, sender, to, content, msg_type="message", extra=None):
        msg = {"type": msg_type, "from": sender,
               "content": content, "timestamp": time.time()}
        if extra:
            msg.update(extra)
        with open(self.dir / f"{to}.jsonl", "a") as f:
            f.write(json.dumps(msg) + "\n")

    def read_inbox(self, name):
        path = self.dir / f"{name}.jsonl"
        if not path.exists(): return "[]"
        msgs = [json.loads(l) for l in path.read_text().strip().splitlines() if l]
        path.write_text("")  # drain
        return json.dumps(msgs, indent=2)
```

4. 每个队友在每次 LLM 调用前检查收件箱, 将消息注入上下文。

```python
def _teammate_loop(self, name, role, prompt):
    messages = [{"role": "user", "content": prompt}]
    for _ in range(50):
        inbox = BUS.read_inbox(name)
        if inbox != "[]":
            messages.append({"role": "user",
                "content": f"<inbox>{inbox}</inbox>"})
            messages.append({"role": "assistant",
                "content": "Noted inbox messages."})
        response = client.messages.create(...)
        if response.stop_reason != "tool_use":
            break
        # execute tools, append results...
    self._find_member(name)["status"] = "idle"
```

5. `broadcast()` 向所有队友群发消息, 本质上是遍历名册对每个成员调一次 `send()`。

```python
def broadcast(self, sender, content):
    for member in self.config["members"]:
        if member["name"] != sender:
            BUS.send(sender, member["name"], content, msg_type="broadcast")
    # 也发给 lead
    BUS.send(sender, "lead", content, msg_type="broadcast")
```

## 设计取舍与局限性

**drain-on-read 的竞态问题**: `read_inbox()` 里先 `read_text()` 再 `write_text("")` 清空, 这两步不是原子操作。如果两个线程同时读同一个收件箱, 可能丢消息。当前设计下每个收件箱只有对应的队友自己会读, 所以实际上不会冲突, 但如果扩展为多消费者模型就需要加文件锁 (`fcntl.flock`) 或换用线程安全的队列。

**无消息确认机制**: 消息发出去就完了, 没有 ACK。发送方不知道对方有没有收到、有没有处理。对于这个教学场景够用, 生产环境需要加确认和重试。

**内存与上下文膨胀**: 每次收到收件箱消息都会追加到 `messages` 列表里, 没有做上下文压缩 (s06)。如果队友之间频繁通信, 上下文会快速膨胀直到超出 token 限制。

**单机限制**: 线程模型意味着所有队友跑在同一个进程里。如果要跨机器分布, 需要把 JSONL 文件换成真正的消息队列 (Redis, RabbitMQ 等)。

## 相对 s08 的变更

| 组件           | 之前 (s08)       | 之后 (s09)                         |
|----------------|------------------|------------------------------------|
| Tools          | 6                | 9 (+spawn/send/read_inbox)         |
| 智能体数量     | 单一             | 领导 + N 个队友                    |
| 持久化         | 无               | config.json + JSONL 收件箱         |
| 线程           | 后台命令         | 每线程完整 agent loop              |
| 生命周期       | 一次性           | idle -> working -> idle            |
| 通信           | 无               | message + broadcast                |

## 试一试

```sh
cd learn-claude-code
python agents/s09_agent_teams.py
```

试试这些 prompt (英文 prompt 对 LLM 效果更好, 也可以用中文):

1. `Spawn alice (coder) and bob (tester). Have alice send bob a message.`
2. `Broadcast "status update: phase 1 complete" to all teammates`
3. `Check the lead inbox for any messages`
4. 输入 `/team` 查看团队名册和状态
5. 输入 `/inbox` 手动检查领导的收件箱
