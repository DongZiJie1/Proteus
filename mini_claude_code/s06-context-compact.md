# s06: Context Compact (上下文压缩)

`s01 > s02 > s03 > s04 > s05 > [ s06 ] | s07 > s08 > s09 > s10 > s11 > s12`

> *"上下文总会满, 要有办法腾地方"* -- 三层压缩策略, 换来无限会话。

## 问题

上下文窗口是有限的。读一个 1000 行的文件就吃掉 ~4000 token; 读 30 个文件、跑 20 条命令, 轻松突破 100k token。不压缩, 智能体根本没法在大项目里干活。

## 解决方案

三层压缩, 激进程度递增:

```
Every turn:
+------------------+
| Tool call result |
+------------------+
        |
        v
[Layer 1: micro_compact]        (silent, every turn)
  Replace tool_result > 3 turns old
  with "[Previous: used {tool_name}]"
        |
        v
[Check: tokens > 50000?]
   |               |
   no              yes
   |               |
   v               v
continue    [Layer 2: auto_compact]
              Save transcript to .transcripts/
              LLM summarizes conversation.
              Replace all messages with [summary].
                    |
                    v
            [Layer 3: compact tool]
              Model calls compact explicitly.
              Same summarization as auto_compact.
```

## 工作原理

1. **第一层 -- micro_compact**: 每次 LLM 调用前, 将旧的 tool result 替换为占位符。

```python
def micro_compact(messages: list) -> list:
    tool_results = []
    for i, msg in enumerate(messages):
        if msg["role"] == "user" and isinstance(msg.get("content"), list):
            for j, part in enumerate(msg["content"]):
                if isinstance(part, dict) and part.get("type") == "tool_result":
                    tool_results.append((i, j, part))
    if len(tool_results) <= KEEP_RECENT:
        return messages
    for _, _, part in tool_results[:-KEEP_RECENT]:
        if len(part.get("content", "")) > 100:
            part["content"] = f"[Previous: used {tool_name}]"
    return messages
```

2. **第二层 -- auto_compact**: token 超过阈值时, 保存完整对话到磁盘, 让 LLM 做摘要。

```python
def auto_compact(messages: list) -> list:
    # Save transcript for recovery
    transcript_path = TRANSCRIPT_DIR / f"transcript_{int(time.time())}.jsonl"
    with open(transcript_path, "w") as f:
        for msg in messages:
            f.write(json.dumps(msg, default=str) + "\n")
    # LLM summarizes
    response = client.messages.create(
        model=MODEL,
        messages=[{"role": "user", "content":
            "Summarize this conversation for continuity..."
            + json.dumps(messages, default=str)[:80000]}],
        max_tokens=2000,
    )
    return [
        {"role": "user", "content": f"[Compressed]\n\n{response.content[0].text}"},
        {"role": "assistant", "content": "Understood. Continuing."},
    ]
```

3. **第三层 -- manual compact**: `compact` 工具按需触发同样的摘要机制。

4. 循环整合三层:

```python
def agent_loop(messages: list):
    while True:
        micro_compact(messages)                        # Layer 1
        if estimate_tokens(messages) > THRESHOLD:
            messages[:] = auto_compact(messages)       # Layer 2
        response = client.messages.create(...)
        # ... tool execution ...
        if manual_compact:
            messages[:] = auto_compact(messages)       # Layer 3
```

完整历史通过 transcript 保存在磁盘上。信息没有真正丢失, 只是移出了活跃上下文。

## 相对 s05 的变更

| 组件           | 之前 (s05)       | 之后 (s06)                     |
|----------------|------------------|--------------------------------|
| Tools          | 5                | 5 (基础 + compact)             |
| 上下文管理     | 无               | 三层压缩                       |
| Micro-compact  | 无               | 旧结果 -> 占位符               |
| Auto-compact   | 无               | token 阈值触发                 |
| Transcripts    | 无               | 保存到 .transcripts/           |

## 试一试

```sh
cd learn-claude-code
python agents/s06_context_compact.py
```

试试这些 prompt (英文 prompt 对 LLM 效果更好, 也可以用中文):

1. `Read every Python file in the agents/ directory one by one` (观察 micro-compact 替换旧结果)
2. `Keep reading files until compression triggers automatically`
3. `Use the compact tool to manually compress the conversation`


## 深入理解：三层压缩的设计哲学

### 为什么是三层而不是一层？

一个常见的疑问：直接做摘要不就行了，为什么要分三层？

原因是**压缩有代价**。每次摘要都要调用 LLM，既花时间又花钱，而且摘要必然丢失细节。三层设计的核心思想是**能省则省，能晚则晚**：

| 层级 | 代价 | 信息损失 | 触发频率 |
|------|------|----------|----------|
| micro_compact | 几乎为零（字符串替换） | 低（只丢旧 tool 输出的细节） | 每轮 |
| auto_compact | 高（一次 LLM 调用） | 中（整个对话压成摘要） | 偶尔 |
| manual compact | 高（一次 LLM 调用） | 中 | 极少 |

micro_compact 能处理 80% 的膨胀问题（tool result 是 token 大户），只有它兜不住时才上更重的手段。

### KEEP_RECENT 和 THRESHOLD 的选择

```python
KEEP_RECENT = 3      # micro_compact 保留最近 3 个 tool result
THRESHOLD = 50000    # auto_compact 在 50k token 时触发
```

- `KEEP_RECENT = 3`：模型通常只需要最近几轮的完整 tool 输出来保持连贯性。太大浪费 token，太小会让模型"失忆"刚做过的事。
- `THRESHOLD = 50000`：留出足够空间给模型生成回复（大多数模型上下文窗口 128k-200k）。设太高会导致模型在窗口末端性能下降，设太低会频繁触发摘要。

### Token 估算的实现

文档中的 `estimate_tokens` 没有展开，实际实现通常有两种方式：

```python
# 方式一：粗略估算（快，不精确）
def estimate_tokens(messages: list) -> int:
    text = json.dumps(messages, default=str)
    return len(text) // 4  # 英文约 4 字符/token，中文约 2 字符/token

# 方式二：使用 tokenizer（慢，精确）
import tiktoken
enc = tiktoken.encoding_for_model("gpt-4")  # 或 anthropic 的 tokenizer
def estimate_tokens(messages: list) -> int:
    text = json.dumps(messages, default=str)
    return len(enc.encode(text))
```

实际工程中通常用方式一，因为精确计数不值得为每轮循环付出额外延迟。

### Transcript 的恢复机制

transcript 不只是备份，它还支持**会话恢复**。如果智能体崩溃或用户重新打开项目，可以从 transcript 重建上下文：

```python
def recover_from_transcript(transcript_path: Path) -> list:
    messages = []
    with open(transcript_path) as f:
        for line in f:
            messages.append(json.loads(line))
    # 恢复后立即做一次压缩，避免重新撑爆上下文
    return auto_compact(messages)
```

这就是为什么 auto_compact 第一步是存 transcript —— 它既是安全网，也是恢复入口。

### 摘要 Prompt 的设计要点

auto_compact 中的摘要 prompt 看似简单，实际上需要精心设计：

```python
COMPACT_PROMPT = """Summarize this conversation for continuity.
Focus on:
1. What task the user is working on (goal)
2. What has been accomplished so far (progress)
3. What files have been read or modified (artifacts)
4. Any decisions or preferences expressed (context)
5. What remains to be done (next steps)

Do NOT include tool call details or file contents.
Keep the summary under 2000 tokens."""
```

关键是告诉 LLM **保留什么、丢弃什么**。目标、进度、产物、决策、下一步 —— 这五类信息是模型继续工作的最小必要上下文。tool 调用细节和文件内容是最大的 token 消耗者，也是最容易从磁盘重新获取的，所以优先丢弃。

### 与 s05 Skill Loading 的协同

压缩后，之前通过 `load_skill` 加载的技能内容也会被压缩掉。但这不是问题 —— 模型在摘要中会记住"我加载了 git skill"，需要时再调用一次 `load_skill` 重新加载即可。这也是为什么 s05 的技能设计为按需加载而非写死在 system prompt 里：**可丢弃、可重载**的设计天然适配上下文压缩。

### 与真实产品的对比

| 特性 | 本教程 (s06) | Claude Code (真实) | Cursor / Windsurf |
|------|-------------|-------------------|-------------------|
| 旧结果替换 | micro_compact | 类似机制 | 有，细节不同 |
| 自动摘要 | 50k 阈值 | 动态阈值 | 有，策略不公开 |
| 手动压缩 | compact 工具 | `/compact` 命令 | 通常自动处理 |
| 历史保存 | .transcripts/ | 内部存储 | 内部存储 |
| 摘要模型 | 同一个模型 | 可能用更小的模型 | 不公开 |

真实产品的核心思路和 s06 一致，区别主要在工程细节：阈值动态调整、摘要用更便宜的模型、更精细的 token 计数等。
