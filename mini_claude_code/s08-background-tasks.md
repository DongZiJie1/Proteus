# s08: Background Tasks (后台任务)

`s01 > s02 > s03 > s04 > s05 > s06 | s07 > [ s08 ] s09 > s10 > s11 > s12`

> *"慢操作丢后台, agent 继续想下一步"* -- 后台线程跑命令, 完成后注入通知。

## 问题

有些命令要跑好几分钟: `npm install`、`pytest`、`docker build`。阻塞式循环下模型只能干等。用户说 "装依赖, 顺便建个配置文件", 智能体却只能一个一个来。

## 解决方案

```
Main thread                Background thread
+-----------------+        +-----------------+
| agent loop      |        | subprocess runs |
| ...             |        | ...             |
| [LLM call] <---+------- | enqueue(result) |
|  ^drain queue   |        +-----------------+
+-----------------+

Timeline:
Agent --[spawn A]--[spawn B]--[other work]----
             |          |
             v          v
          [A runs]   [B runs]      (parallel)
             |          |
             +-- results injected before next LLM call --+
```

## 工作原理

1. BackgroundManager 用线程安全的通知队列追踪任务。

```python
class BackgroundManager:
    def __init__(self):
        self.tasks = {}
        self._notification_queue = []
        self._lock = threading.Lock()
```

2. `run()` 启动守护线程, 立即返回。

```python
def run(self, command: str) -> str:
    task_id = str(uuid.uuid4())[:8]
    self.tasks[task_id] = {"status": "running", "command": command}
    thread = threading.Thread(
        target=self._execute, args=(task_id, command), daemon=True)
    thread.start()
    return f"Background task {task_id} started"
```

3. 子进程完成后, 结果进入通知队列。

```python
def _execute(self, task_id, command):
    try:
        r = subprocess.run(command, shell=True, cwd=WORKDIR,
            capture_output=True, text=True, timeout=300)
        output = (r.stdout + r.stderr).strip()[:50000]
    except subprocess.TimeoutExpired:
        output = "Error: Timeout (300s)"
    with self._lock:
        self._notification_queue.append({
            "task_id": task_id, "result": output[:500]})
```

4. 每次 LLM 调用前排空通知队列。

```python
def agent_loop(messages: list):
    while True:
        notifs = BG.drain_notifications()
        if notifs:
            notif_text = "\n".join(
                f"[bg:{n['task_id']}] {n['result']}" for n in notifs)
            messages.append({"role": "user",
                "content": f"<background-results>\n{notif_text}\n"
                           f"</background-results>"})
            messages.append({"role": "assistant",
                "content": "Noted background results."})
        response = client.messages.create(...)
```

循环保持单线程。只有子进程 I/O 被并行化。

## 相对 s07 的变更

| 组件           | 之前 (s07)       | 之后 (s08)                         |
|----------------|------------------|------------------------------------|
| Tools          | 8                | 6 (基础 + background_run + check)  |
| 执行方式       | 仅阻塞           | 阻塞 + 后台线程                    |
| 通知机制       | 无               | 每轮排空的队列                     |
| 并发           | 无               | 守护线程                           |

## 试一试

```sh
cd learn-claude-code
python agents/s08_background_tasks.py
```

试试这些 prompt (英文 prompt 对 LLM 效果更好, 也可以用中文):

1. `Run "sleep 5 && echo done" in the background, then create a file while it runs`
2. `Start 3 background tasks: "sleep 2", "sleep 4", "sleep 6". Check their status.`
3. `Run pytest in the background and keep working on other things`

## 补充：厨师与灶台 —— 一个直觉类比

把 agent 想象成一个厨师（单线程），厨房里有好几个灶台（后台线程）。

- **s07 的问题**：炖一锅汤要 30 分钟，厨师站在锅前傻等，客人说"炖汤的时候顺便切个菜"，厨师说"不行，等汤好了才能动"。
- **s08 的做法**：汤丢到灶台上炖着，厨师转身去切菜炒别的。灶台上的汤好了会"叮"一声（通知队列），厨师听到了再去处理。

`drain_notifications()` 就是"回头看一眼灶台上的汤好了没"的动作。

## 补充：后台任务还没跑完时会怎样？

agent 不会等。`drain_notifications()` 只拿队列里已有的结果，队列空就跳过，LLM 照常调用。

```
第1轮: spawn "npm install" → drain → 队列空 → LLM正常回答 → 去建配置文件
第2轮: drain → 队列还是空(还在装) → LLM正常回答 → 继续干活
第3轮: drain → 队列有了(装完了) → 结果注入给LLM → LLM知道装好了
```

agent 永远不会被卡住。最坏情况就是"还没好，先不管"。

## 补充：边界情况 —— 活干完了，后台任务还没好

如果 LLM 判断所有前台工作都做完了，不再发 tool call，agent loop 就会 `break` 退出。此时后台任务可能还在跑。

结果最终会被塞进队列，但已经没人 drain 了 —— **厨师下班走人了，灶台上的汤好了，但厨房没人了。**

结果不会彻底丢失：`BackgroundManager` 还活着，`tasks` 字典里记着状态。如果用户再发一条消息触发新的 agent loop，第一次 drain 就能拿到之前的结果。但如果用户不再说话，结果就静静躺在队列里无人处理。

这是 s08 作为教学示例的简化取舍。生产系统通常会：
- 后台任务全部完成前不退出 agent loop
- 任务完成后主动推送通知给用户
- 设置超时兜底机制
