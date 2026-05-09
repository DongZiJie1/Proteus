# s07: Task System (任务系统)

`s01 > s02 > s03 > s04 > s05 > s06 | [ s07 ] s08 > s09 > s10 > s11 > s12`

> *"大目标要拆成小任务, 排好序, 记在磁盘上"* -- 文件持久化的任务图, 为多 agent 协作打基础。

## 问题

s03 的 TodoManager 只是内存中的扁平清单: 没有顺序、没有依赖、状态只有做完没做完。真实目标是有结构的 -- 任务 B 依赖任务 A, 任务 C 和 D 可以并行, 任务 E 要等 C 和 D 都完成。

没有显式的关系, 智能体分不清什么能做、什么被卡住、什么能同时跑。而且清单只活在内存里, 上下文压缩 (s06) 一跑就没了。

## 解决方案

把扁平清单升级为持久化到磁盘的**任务图**。每个任务是一个 JSON 文件, 有状态、前置依赖 (`blockedBy`) 和后置依赖 (`blocks`)。任务图随时回答三个问题:

- **什么可以做?** -- 状态为 `pending` 且 `blockedBy` 为空的任务。
- **什么被卡住?** -- 等待前置任务完成的任务。
- **什么做完了?** -- 状态为 `completed` 的任务, 完成时自动解锁后续任务。

```
.tasks/
  task_1.json  {"id":1, "status":"completed"}
  task_2.json  {"id":2, "blockedBy":[1], "status":"pending"}
  task_3.json  {"id":3, "blockedBy":[1], "status":"pending"}
  task_4.json  {"id":4, "blockedBy":[2,3], "status":"pending"}

任务图 (DAG):
                 +----------+
            +--> | task 2   | --+
            |    | pending  |   |
+----------+     +----------+    +--> +----------+
| task 1   |                          | task 4   |
| completed| --> +----------+    +--> | blocked  |
+----------+     | task 3   | --+     +----------+
                 | pending  |
                 +----------+

顺序:   task 1 必须先完成, 才能开始 2 和 3
并行:   task 2 和 3 可以同时执行
依赖:   task 4 要等 2 和 3 都完成
状态:   pending -> in_progress -> completed
```

这个任务图是 s07 之后所有机制的协调骨架: 后台执行 (s08)、多 agent 团队 (s09+)、worktree 隔离 (s12) 都读写这同一个结构。

## 工作原理

1. **TaskManager**: 每个任务一个 JSON 文件, CRUD + 依赖图。

```python
class TaskManager:
    def __init__(self, tasks_dir: Path):
        self.dir = tasks_dir
        self.dir.mkdir(exist_ok=True)
        self._next_id = self._max_id() + 1

    def create(self, subject, description=""):
        task = {"id": self._next_id, "subject": subject,
                "status": "pending", "blockedBy": [],
                "blocks": [], "owner": ""}
        self._save(task)
        self._next_id += 1
        return json.dumps(task, indent=2)
```

2. **依赖解除**: 完成任务时, 自动将其 ID 从其他任务的 `blockedBy` 中移除, 解锁后续任务。

```python
def _clear_dependency(self, completed_id):
    for f in self.dir.glob("task_*.json"):
        task = json.loads(f.read_text())
        if completed_id in task.get("blockedBy", []):
            task["blockedBy"].remove(completed_id)
            self._save(task)
```

3. **状态变更 + 依赖关联**: `update` 处理状态转换和依赖边。

```python
def update(self, task_id, status=None,
           add_blocked_by=None, add_blocks=None):
    task = self._load(task_id)
    if status:
        task["status"] = status
        if status == "completed":
            self._clear_dependency(task_id)
    self._save(task)
```

4. 四个任务工具加入 dispatch map。

```python
TOOL_HANDLERS = {
    # ...base tools...
    "task_create": lambda **kw: TASKS.create(kw["subject"]),
    "task_update": lambda **kw: TASKS.update(kw["task_id"], kw.get("status")),
    "task_list":   lambda **kw: TASKS.list_all(),
    "task_get":    lambda **kw: TASKS.get(kw["task_id"]),
}
```

从 s07 起, 任务图是多步工作的默认选择。s03 的 Todo 仍可用于单次会话内的快速清单。

5. **list_all 和 get**: 查询接口, 返回全部任务或单个任务的 JSON。

```python
def list_all(self):
    tasks = []
    for f in sorted(self.dir.glob("task_*.json")):
        tasks.append(json.loads(f.read_text()))
    return json.dumps(tasks, indent=2)

def get(self, task_id):
    task = self._load(task_id)
    return json.dumps(task, indent=2)
```

6. **依赖边的双向维护**: `add_blocked_by` 和 `add_blocks` 在 update 中同时维护正向和反向边。

```python
def update(self, task_id, status=None,
           add_blocked_by=None, add_blocks=None):
    task = self._load(task_id)
    if status:
        task["status"] = status
        if status == "completed":
            self._clear_dependency(task_id)
    if add_blocked_by:
        task["blockedBy"].extend(add_blocked_by)
        # 反向: 在前置任务的 blocks 里加上自己
        for dep_id in add_blocked_by:
            dep = self._load(dep_id)
            if task_id not in dep["blocks"]:
                dep["blocks"].append(task_id)
                self._save(dep)
    if add_blocks:
        task["blocks"].extend(add_blocks)
        for dep_id in add_blocks:
            dep = self._load(dep_id)
            if task_id not in dep["blockedBy"]:
                dep["blockedBy"].append(task_id)
                self._save(dep)
    self._save(task)
    return json.dumps(task, indent=2)
```

## 相对 s06 的变更

| 组件 | 之前 (s06) | 之后 (s07) |
|---|---|---|
| Tools | 5 | 8 (`task_create/update/list/get`) |
| 规划模型 | 扁平清单 (仅内存) | 带依赖关系的任务图 (磁盘) |
| 关系 | 无 | `blockedBy` + `blocks` 边 |
| 状态追踪 | 做完没做完 | `pending` -> `in_progress` -> `completed` |
| 持久化 | 压缩后丢失 | 压缩和重启后存活 |

## 试一试

```sh
cd learn-claude-code
python agents/s07_task_system.py
```

试试这些 prompt (英文 prompt 对 LLM 效果更好, 也可以用中文):

1. `Create 3 tasks: "Setup project", "Write code", "Write tests". Make them depend on each other in order.`
2. `List all tasks and show the dependency graph`
3. `Complete task 1 and then list tasks to see task 2 unblocked`
4. `Create a task board for refactoring: parse -> transform -> emit -> test, where transform and emit can run in parallel after parse`


## 深入理解：从扁平清单到任务图

### Todo (s03) vs TaskManager (s07) -- 什么时候用哪个？

两者不是替代关系, 而是不同粒度的工具:

| 维度 | Todo (s03) | TaskManager (s07) |
|------|-----------|-------------------|
| 存储 | 内存 | 磁盘 (JSON 文件) |
| 生命周期 | 当前会话 | 跨会话、跨重启 |
| 结构 | 扁平列表 | DAG (有向无环图) |
| 依赖 | 无 | `blockedBy` + `blocks` |
| 适用场景 | "重构这个文件" (3-5 步) | "搭建整个微服务" (20+ 步) |
| 并行 | 不支持 | 天然支持 |


### 为什么每个任务一个文件, 而不是一个大 JSON？

```
方案 A: 单文件                    方案 B: 每任务一个文件 (采用)
.tasks/all.json                  .tasks/task_1.json
[{id:1,...}, {id:2,...}, ...]    .tasks/task_2.json
                                 .tasks/task_3.json
```

原因有三:

1. **并发安全**: s08 后台任务和 s09 多 agent 会同时读写任务。单文件需要文件锁, 多文件天然隔离 -- 两个 agent 同时更新 task_2 和 task_3 不会冲突。
2. **原子性**: 写一个小文件几乎是原子操作, 崩溃时最多丢一个任务的最新状态, 不会损坏整个任务图。
3. **增量读取**: `list_all` 可以只读需要的文件, 不用反序列化整个大 JSON。

这个设计为后续的多 agent 并发 (s09) 埋下了伏笔。

### DAG 而非树: 为什么 blockedBy 是数组？

任务 4 的 `blockedBy: [2, 3]` 表示它有两个前置依赖。这是 DAG (有向无环图) 而非树的关键特征 -- 一个节点可以有多个父节点。

```
树结构 (不够用):              DAG (采用):
    1                            1
   / \                          / \
  2   3                        2   3
  |   |                         \ /
  4   5                          4
```

树结构下, task 4 只能依赖一个父节点。但现实中 "集成测试" 要等 "前端完成" 和 "后端完成" 两个任务都做完, 这就是多依赖。

### 环检测: 为什么叫 "无环" 图？

如果 task A 依赖 B, B 依赖 C, C 又依赖 A, 就形成了环 -- 没有任何任务能开始。当前实现没有显式的环检测, 这是一个有意的简化。原因:

- 任务由 LLM 创建, 通常不会故意制造环
- 即使出现环, 表现为 "所有任务都 blocked", 模型会注意到并修复
- 完整的环检测 (拓扑排序) 增加复杂度, 在教学代码中不值得

真实产品中会加入环检测, 在 `add_blocked_by` 时做一次 DFS 判断是否会形成环。

### _clear_dependency 的性能问题

```python
def _clear_dependency(self, completed_id):
    for f in self.dir.glob("task_*.json"):  # 遍历所有文件
        task = json.loads(f.read_text())
        if completed_id in task.get("blockedBy", []):
            task["blockedBy"].remove(completed_id)
            self._save(task)
```

每次完成一个任务, 都要遍历所有任务文件。10 个任务没问题, 1000 个就慢了。优化方向:

- **用 `blocks` 字段反查**: 完成 task 1 时, 直接读 task 1 的 `blocks: [2, 3]`, 只更新 task 2 和 task 3, 不用遍历全部。
- **内存索引**: 启动时建一个 `{task_id: [blocked_tasks]}` 的字典, 避免每次都读磁盘。

当前实现选择了最简单的全遍历, 因为教学代码优先可读性。

### owner 字段: 为 s09 多 agent 预留

```python
task = {"id": self._next_id, "subject": subject,
        "status": "pending", "blockedBy": [],
        "blocks": [], "owner": ""}
```

`owner` 在 s07 中始终为空字符串, 但它是为 s09 多 agent 团队准备的。到时候:

- `owner: "frontend-agent"` 表示这个任务分配给了前端 agent
- `owner: ""` 表示任务还没被认领
- 多个 agent 通过 `owner` 字段实现任务分配, 避免重复工作

这就是为什么任务图是 "协调骨架" -- 它不只是记录任务, 还记录谁在做什么。

### 状态机: 三态 vs 两态

s03 的 Todo 只有两态: `pending` 和 `completed`。s07 加了 `in_progress`:

```
s03:  pending ---------> completed

s07:  pending --> in_progress --> completed
```

`in_progress` 的意义:
- 让其他 agent 知道 "这个任务有人在做了, 别抢"
- 区分 "还没开始" 和 "正在做" -- 对进度追踪很重要
- 配合 `owner` 字段, 实现任务认领机制

### 与 s06 上下文压缩的协同

s06 的压缩会清掉内存中的对话历史。如果任务计划只存在 messages 里, 压缩后就丢了。s07 把任务持久化到磁盘, 压缩后模型只需要调用 `task_list` 就能恢复全部任务状态。

```
压缩前: messages 里有完整的任务讨论过程
压缩后: messages 被摘要替换, 但 .tasks/ 目录完好
恢复:   模型调用 task_list, 立刻知道哪些做了、哪些没做、哪些被卡住
```

这是 s07 解决的最核心问题 -- 任务状态不再依赖上下文窗口的生存。

### 与真实产品的对比

| 特性 | 本教程 (s07) | Claude Code | Cursor / Windsurf |
|------|-------------|-------------|-------------------|
| 任务存储 | JSON 文件 | 内部状态 + TodoManager | 不公开 |
| 依赖关系 | blockedBy/blocks | 扁平清单 (无显式依赖) | 不公开 |
| 持久化 | .tasks/ 目录 | 会话内存 | 不公开 |
| 并行支持 | DAG 天然支持 | 无 (单线程) | 部分支持 |
| 多 agent | owner 字段预留 | 无 | 无 |

Claude Code 目前仍使用类似 s03 的扁平 TodoManager, 没有显式的依赖图。s07 的设计更接近项目管理工具 (如 Linear、Jira) 的任务模型, 是为多 agent 场景设计的。
