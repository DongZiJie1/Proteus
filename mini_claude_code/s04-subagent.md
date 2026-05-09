# s04: Subagents (子智能体)

`s01 > s02 > s03 > [ s04 ] s05 > s06 | s07 > s08 > s09 > s10 > s11 > s12`

> *"大任务拆小, 每个小任务干净的上下文"* -- 子智能体用独立 messages[], 不污染主对话。

## 问题

智能体工作越久, messages 数组越胖。每次读文件、跑命令的输出都永久留在上下文里。"这个项目用什么测试框架?" 可能要读 5 个文件, 但父智能体只需要一个词: "pytest。"

## 解决方案

```
Parent agent                     Subagent
+------------------+             +------------------+
| messages=[...]   |             | messages=[]      | <-- fresh
|                  |  dispatch   |                  |
| tool: task       | ----------> | while tool_use:  |
|   prompt="..."   |             |   call tools     |
|                  |  summary    |   append results |
|   result = "..." | <---------- | return last text |
+------------------+             +------------------+

Parent context stays clean. Subagent context is discarded.
```

## 工作原理

1. 父智能体有一个 `task` 工具。子智能体拥有除 `task` 外的所有基础工具 (禁止递归生成)。

```python
PARENT_TOOLS = CHILD_TOOLS + [
    {"name": "task",
     "description": "Spawn a subagent with fresh context.",
     "input_schema": {
         "type": "object",
         "properties": {"prompt": {"type": "string"}},
         "required": ["prompt"],
     }},
]
```

2. 子智能体以 `messages=[]` 启动, 运行自己的循环。只有最终文本返回给父智能体。

```python
def run_subagent(prompt: str) -> str:
    sub_messages = [{"role": "user", "content": prompt}]
    for _ in range(30):  # safety limit
        response = client.messages.create(
            model=MODEL, system=SUBAGENT_SYSTEM,
            messages=sub_messages,
            tools=CHILD_TOOLS, max_tokens=8000,
        )
        sub_messages.append({"role": "assistant",
                             "content": response.content})
        if response.stop_reason != "tool_use":
            break
        results = []
        for block in response.content:
            if block.type == "tool_use":
                handler = TOOL_HANDLERS.get(block.name)
                output = handler(**block.input)
                results.append({"type": "tool_result",
                    "tool_use_id": block.id,
                    "content": str(output)[:50000]})
        sub_messages.append({"role": "user", "content": results})
    return "".join(
        b.text for b in response.content if hasattr(b, "text")
    ) or "(no summary)"
```

子智能体可能跑了 30+ 次工具调用, 但整个消息历史直接丢弃。父智能体收到的只是一段摘要文本, 作为普通 `tool_result` 返回。

## 相对 s03 的变更

| 组件           | 之前 (s03)       | 之后 (s04)                    |
|----------------|------------------|-------------------------------|
| Tools          | 5                | 5 (基础) + task (仅父端)      |
| 上下文         | 单一共享         | 父 + 子隔离                   |
| Subagent       | 无               | `run_subagent()` 函数         |
| 返回值         | 不适用           | 仅摘要文本                    |

## 试一试

```sh
cd learn-claude-code
python agents/s04_subagent.py
```

试试这些 prompt (英文 prompt 对 LLM 效果更好, 也可以用中文):

1. `Use a subtask to find what testing framework this project uses`
2. `Delegate: read all .py files and summarize what each one does`
3. `Use a task to create a new module, then verify it from here`


## 补充理解

### 核心问题：上下文污染

智能体每次读文件、执行命令的结果都会永久留在 `messages[]` 里。任务越多，数组越大，最终撑爆上下文窗口。而很多中间过程的信息，父智能体根本不需要。

类比：你是项目经理，不需要知道员工每一步怎么做的，只需要最终报告。

### 两个 Loop 的嵌套关系

主智能体和子智能体本质上是同一种结构，都是一个 agent loop：

```
while True:
    response = 调用 LLM
    if 没有 tool_use:
        break  # 结束
    执行工具
    把结果塞回 messages
```

区别只有一个：
- 主智能体的 loop 贯穿整个对话，messages 一直累积
- 子智能体的 loop 是一次性的，结束后整个 messages 直接丢掉，只把最后的文字返回给主智能体

```
主 loop 跑着跑着...
    → 遇到需要子任务
    → 启动子 loop（全新 messages=[]）
        子 loop 跑完...
        → 返回一句摘要
    → 主 loop 继续，收到的只是那句摘要
主 loop 继续跑...
```

本质：一个 loop 内部嵌套调用了另一个 loop，但内层 loop 的状态不会泄漏到外层。

### 具体例子

问父智能体："这个项目用什么测试框架？"

- 没有子智能体：父智能体自己读 `requirements.txt`、`setup.py`、`pyproject.toml`... 每个文件内容都留在 messages 里
- 有子智能体：派出子智能体去查，子智能体读了 5 个文件，最后只返回 "pytest"，父智能体的 messages 里只多了这一个词

子智能体 = 一次性的上下文沙箱，用完即扔。
