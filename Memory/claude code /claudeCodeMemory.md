# Claude Code 架构分析

> 本文档通过对 Claude Code 架构分析图片的 OCR 提取整理而成，保持原文内容不变。

---

## 4.4.2.1 四层记忆存储

指令文件其实在CC的核心用法就讲过，这里Claude.md 系列文件的维护，需要人工维护的。

### 1. 指令文件 (CLAUDE.md 族) —— 人写的规则

唯一由人手动编写的记忆，告诉 Claude 项目的规则和约束。分 4 个层级:

- **管理员全局**: `/etc/claude-code/CLAUDE.md` —— IT 管理员设定的全局规则
- **用户全局**: `~/.claude/CLAUDE.md` —— 个人偏好 (跨项目生效)
- **项目级**: `CLAUDE.md`、`.claude/CLAUDE.md`、`.claude/rules/*.md` —— 随 Git 提交，团队共享
- **本地私有**: `CLAUDE.local.md` —— 不提交 Git，个人使用

支持 `include` 指令引用其他文件。上限 40,000 字符。Boris (Claude Code 工程负责人) 建议写法是短、有主见、可操作——不是项目历史小说，而是决策规则和约束。

### 2. 对话上下文 (messages[]) —— 短期记忆

就是大模型多轮对话的数组，每个Agent都有。

当前对话的消息数组，所有 LLM 应用都有。Claude Code 的特别之处在于配套了一套多层压缩策略来管理它 (见 3.1.3) 。

这个就是随着每次会话记录的，ClaudeCode特有设计。

### 3. Session Memory —— 当前会话的结构化笔记

一个后台子代理 (Fork Agent) 在你工作的同时持续维护的结构化笔记文件。不是上下文压缩，而是独立的信息提取。内容包括: 任务标题和描述、当前状态、涉及的文件列表、遇到的错误和解决方案、经验教训、工作日志。

触发条件: 对话 token 超过 10K 后启动，之后每 5K tokens 更新一次。后台执行，不阻塞工作。

它的价值:
- `/resume` 恢复会话时，从这个文件快速恢复上下文，无需重新喂全部历史
- 上下文压缩时可作为摘要来源 (见 3.1.3 第 3 层)
- `/skillify` 可以将其转化为可复用的 SKILL.md 文件

---

## 4. 持久记忆 (Memdir) —— 跨会话的长期记忆

最核心的长期记忆系统，位于 `~/.claude/memory/` 。存 4 类信息:

- **user**: 用户角色、偏好、技能水平。例: "偏好函数式风格"、"是后端工程师"
- **feedback**: 对 AI 的纠正和确认。例: "上次用 forEach 被要求改成 map"
- **project**: 项目目标、决策、截止日期。例: "Q3 要迁移到 TypeScript"
- **reference**: 外部系统指针。例: "Bug tracker 在 Linear: xxx"

注意这里故意不存代码事实 (如"函数 X 在第 30 行") ，因为代码会变但记忆不会自动更新，过时的代码记忆会变成危险的误导。代码相关的事实永远用 grep/glob 实时搜索，只有人的偏好和判断才值得持久化。

### 存储结构

```
~/.claude/memory/
├── MEMORY.md                 — 索引文件 (每次会话自动加载)
├── user_role.md              — 主题文件: 用户角色
├── feedback_testing.md       — 主题文件: 测试相关反馈
├── project_migration.md      — 主题文件: 迁移计划
└── reference_tracker.md      — 主题文件: 外部系统指针
```

MEMORY.md 是索引，不存记忆内容本身——每行只有一个指向主题文件的链接和一句话摘要 (<150 字符) ，格式如 `- [用户角色](user_role.md) - 后端工程师，偏好函数式风格` 。上限 200 行/25KB。

实际的记忆内容存在各个主题文件中，按语义主题命名 (不是固定 4 个文件) ，每个文件带 YAML frontmatter 声明 name, description 和 type (对应上面的 user/feedback/project/reference 四种类型) 。查询时系统通过扫描 frontmatter 来判断哪些文件与当前对话相关。

团队记忆: Memdir 下有 team/ 子目录，支持多用户共享的项目记忆。会话开始时自动同步。敏感信息 (API 密钥等) 禁止存入。Feature flag TEAMMEM 控制。

---

## 4.4.2.2 上下文组装

每轮对话开始时，系统从上述 4 种存储中组装完整的上下文注入 system prompt。

### 指令文件的加载

所有层级的 CLAUDE.md 都会被拼接在一起注入，不会互相覆盖。但拼接顺序有讲究——源码注释写道: "Files are loaded in reverse order of priority, ie. the latest files are highest priority with the model paying more attention to them." 这利用了 LLM 的一个已知特性: 模型对 prompt 末尾的内容天然给予更高注意力 (recency bias) 。因此排在后面的文件在实践中"优先级更高"。

加载顺序 (排在越后面，模型越关注) : 管理员全局 > 用户全局 > 项目级 > 本地私有。举例: 如果管理员规则说"用 npm"，但项目级 CLAUDE.md 说"用 pnpm"，模型会倾向于遵循后者。

### 持久记忆的加载

1. MEMORY.md 前 200 行每次会话自动加载到 system prompt
2. 用 Sonnet (小模型，不是主模型 Opus) 扫描所有记忆文件的标题和 frontmatter 描述，选出最多 5 条最相关的，作为 relevant_memories 附件注入上下文
3. 排除本次会话已召回过的记忆 (避免重复)
4. 超过 1 天的记忆自动加过期警告: "关于以下内容的声明可能已过时，请先验证。"
5. 会话总预算 60KB

值得一提的是，记忆召回没有使用 RAG (检索增强生成) 。Boris 在播客中透露，早期版本曾尝试过本地向量数据库 + 云端 embedding 的 RAG 方案，但遇到了多个问题: 代码修改后索引不同步(新写的函数搜不到) 、索引的权限管控复杂 (谁能访问、如何防止越权) 。最终发现 agentic search (模型自主调用 glob + grep) 全面优于 RAG——Boris 的原话是 "agentic search just outperformed everything, and when I say agentic search, this is a fancy word for glob and grep."

---

## 4.4.2.3 压缩

压缩是Agent里面的一个核心技巧。怎么样压缩保证模型能记住最重要的东西，直接影响接下来对话的好坏。我们可以看到 Claude Code的压缩策略其实非常复杂，是用心在设计的。

对话越来越长，上下文窗口终会被填满。Claude Code 设计了一套 5 层压缩流水线，在每轮查询循环中按固定顺序依次执行，每层有独立的触发条件——不是每层每次都触发，而是逐层检查、按需执行，越往后越激进:

```
第1层 Tool Result Budget      — 每轮都跑，限制单条消息体积
第2层 Microcompact             — 清理旧工具结果内容
第3层 Auto-compact             — 用 LLM 做全量摘要(核心层)
第4层 Blocking Limit           — 上限: 上面都压不下来就报错
第5层 Reactive Compact         — API 返回 413 后的紧急重试
```

第一层是为了压缩工具，如果工具调用结果返回内容太多，就只给索引，按需加载。有点像SKILL的思想，渐进式披露。

### 第 1 层: Tool Result Budget (工具结果预算)

每轮都执行。当你一轮让 Claude 跑了多个并行工具调用 (比如 6 个 grep) ，每个返回几十 KB，单条消息的工具结果累计可能超过 200K 字符 的预算上限。这时系统会把最大的工具结果存到磁盘，原位替换成约 2KB 的预览 + 文件路径。模型后续需要完整内容时，可以用 Read 工具去读磁盘上的完整文件。这本质上是一种**渐进式披露 (Progressive Disclosure)** 的思想——先给模型足够判断的预览，让它自己决定要不要加载全量。很多时候 2KB 预览就够用了，完整文件可能永远不会被读取。

这一层设计就是为了清理工具。多轮调用后，工具的结果会一直跟随在上下文中 (没压缩前) 。这里要解决的核心就是适当处理过期的工具调用结果。

---

### 第 2 层: Microcompact (工具结果清理)

理解这一层需要先了解 prompt Cache 机制: Claude Code 的每次 API 调用都会把完整对话历史 (system prompt + 所有消息) 重新发送给服务端。随着对话进行，这个输入可能达到 100K+ tokens。服务端会缓存这个输入的前缀 (TTL 约 60 分钟) ，后续请求如果前缀相同就直接复用，不需要从头处理——这缓存的是"你发过去的问题"，不是"模型返回的回答"。这意味着: 如果客户端修改了对话历史中的任何旧内容，前缀就变了，缓存失效，服务端要重新处理全部 tokens——更贵、更慢。

这一层要解决的问题就是: 怎么清理旧的工具结果，同时尽量不破坏这个输入缓存。有两个子策略，按优先级尝试，命中一个就停:

1. **基于时间的 Microcompact**: 如果距上次助手回复超过 60 分钟 (缓存 TTL 已自然过期) ，直接将旧的工具结果替换为 '[Old tool result content cleared]' ，只保留最近 5 条。适用的工具类型: Read, Bash, Grep, Glob, WebSearch, WebFetch, Edit, Write 。因为缓存已经失效，直接改内容不会有额外损失。

2. **基于缓存编辑的 Microcompact (Anthropic 内部)**: 客户端消息完全不改，而是额外发一条 cache_edits 指令，告诉服务端"在你缓存的副本里删掉这些工具结果"——prompt 前缀不变，缓存继续有效，省钱。客户端的本地消息数组保持完整 (用于 session 持久化、resume 等) ，但模型实际看到的上下文已经被缩减了。

如果两个都不适用，本层不做任何压缩，把压力交给后面的 Auto-compact。

上面两层都是在上下文窗口还没有很大的时候就尝试压缩，这是Claude Code的巧思。很多Agent都是在上下文快满了才压缩，包括openclaw。第三步，自动压缩也就是所有Agent都会做的，窗口满了，一定要压缩的策略。

---

### 第 3 层: Auto-compact (自动全量压缩) —— 核心层

计算逻辑 (以 200K 上下文窗口为例):

```
有效窗口 = 200,000 - min(maxOutputTokens, 20,000) = 180,000
自动压缩阈值 = 180,000 - 13,000 = 167,000 tokens
```

即当前对话 token 数超过 167K 时自动触发。

触发后有两个子策略，先尝试轻量的，不行再用重量的:

1. **Session Memory Compact (轻量)**: 用已经由后台代理提取好的 Session Memory 文件作为摘要 (见 3.1.1.3) ，保留最近 10K~40K tokens 的原始对话 (至少保留 5 条含文本的消息) 。不需要额外的 LLM 调用，所以很便宜。

2. **Full LLM Compact (重量)**: 把整段对话发给 LLM 做结构化摘要。摘要输出上限 20K tokens，包含 9 个固定段落:
   - 核心请求 (Primary Request)
   - 关键概念 (Key Concepts)
   - 涉及文件列表 (Files)
   - 代码错误与修复 (Errors)
   - 问题解决过程 (Problem Solving)
   - 所有用户消息 (User Messages)
   - 待办事项 (Pending Tasks)
   - 当前工作 (Current Work)
   - 下一步计划 (Next Step)

3. **压缩后自动恢复文件上下文**: 压缩会丢弃旧消息，但会话期间所有被读过、写过、编辑过的文件都记录在 readFileState 缓存中 (包括 agent 调用 FileRead/FileEdit/FileWrite/Bash 工具操作的文件，以及 memdir 系统和 Sonnet 预取自动加载的记忆文件) 。压缩完成后，系统会从这个缓存中排除 CLAUDE.md 和 Plan 文件 (它们有各自的恢复机制) ，再按最近访问时间排序，取最多 5 个文件从磁盘重新读取 (不用旧缓存，确保内容是最新的) ，以每文件 <5K tokens、总计 <50K tokens 的预算注入到压缩后的上下文中，让模型不需要在下一轮重新读取这些文件就能继续工作。如果会话中没有操作过任何文件，则不会恢复。

4. **防御机制**: 连续失败 3 次后触发熔断，不再尝试。如果压缩调用本身也报 prompt-too-long，会从最旧的消息开始截断，最多重试 3 次。

---

### 手动触发

用户可以随时用 `/compact` 命令手动触发 Full LLM Compact，还可以附带提示词指定保留什么——比如 `/compact 重点保留决策和待办事项` 。手动触发的阈值更宽松 (buffer 只留 3K 而不是 13K) 。

### 和 OpenClaw 的对比

OpenClaw 只有一级压缩，但做得更精细: 保留最近 ~20K tokens 原文不动，只压缩更早的旧消息，旧消息按 token 数分成多个块逐块总结，再合并成一个连贯摘要。压缩前还会静默运行一轮，提醒 Agent 把重要内容写入记忆文件 (Pre-Compaction Memory Flush) ，防止关键信息随压缩丢失。压缩后结构: [合并摘要] + [最近 ~20K tokens 原文] 。

Claude Code 则是两级梯度设计; 轻量级的 Session Memory Compact 效果类似 OpenClaw (摘要 + 原文) ，但摘要不是现场生成的，而是后台子代理提前维护好的，所以不需要额外 LLM 调用、更省成本。只有轻量级打不住时才上 Full LLM Compact——这一步更激进，把全部对话压缩成结构化摘要，旧消息全部丢弃。

核心差异: OpenClaw 一步到位但精细 (分块 + 合并 + 预刷记忆) ，Claude Code 分两步走 (先便宜后贵) ，用梯度换成本效率。

下图是 Claude Code 的压缩策略 vs OpenClaw 的压缩策略:

```
Auto-compact: Two Strategies

Claude Code (2-tier gradient)                    OpenClaw (1-tier fine-grained)
                                                              │
Lightweight — Session Memory Compact            Split old messages into chunks
    │                                           逐个块总结 → 合并成一个连贯摘要
    ▼                                                       │
Full LLM Compact                                [Merged Summary] + [recent ~20K]
```

---

### 第 4 层: Blocking Limit (硬拒绝)

如果经过前 3 层之后 token 数仍然超过硬上限 (有效窗口 - 3000，约 177K) ，直接向用户输出 PROMPT_TOO_LONG_ERROR_MESSAGE 并终止本轮。这是发起 API 调用之前的最后一道门。

### 第 5 层: Reactive Compact (事后兜底)

在 API 调用之后触发——当 API 返回 prompt_too_long (413) 或媒体文件过大错误时，拦截错误，对旧消息做一次摘要压缩，然后重试 API 调用。每轮最多尝试 1 次，防止死循环。

### 为什么建议主动 /compact

社区分析源码后的一个关键建议是: 主动使用 `/compact` ，不要等系统自动压缩 (出自 "I Read 512,000 Lines of Leaked Claude Code" 视频) 。原因很简单——自动压缩触发时，你无法控制保留什么，系统按自己的 9 段模板做摘要，可能会丢掉你认为重要的上下文。而手动 `/compact` 可以附带指令，比如 `/compact 重点保留决策和待办事项` ，让 LLM 知道什么该保留。Boris 在播客中也提到自动压缩是实现"本质上无限上下文"(essentially infinite context) 的核心手段，但这更多是系统兜底，用户主动管理上下文效果会更好。

### 总结: 压缩策略的设计哲学

这 5 层形成一个从温和到激进的梯度: 第 1 层每轮都会触发，Full Compact 是中等频率事件，Blocking Limit 和 Reactive Compact 是极少触发的兜底。

---

## 4.4.2.4 记忆写入

上面讲了 4 种存储和加载/压缩策略，但记忆是怎么写进去的? 有 3 个写入机制:

### 1. extractMemories (持久记忆提取)

在每次完整查询循环结束后 (模型给出最终回复、没有更多工具要调用时) ，由一个 Fork Agent 后台执行:
- 继承父对话的完整上下文，共享 prompt cache (不额外消耗 token)
- 严格权限沙箱: 只能读文件 + 只能写记忆目录，不能执行 bash、不能调 MCP、不能启动子代理
- 高效的 2 轮模式: 第 1 轮并行读所有需更新的文件 → 第 2 轮并行写所有变更
- 有频率限制，不是每轮都跑

### 2. Session Memory 更新

对话 token 超过 10K 后启动，之后每 5K tokens 由后台 Fork Agent 更新一次结构化笔记文件。

### 3. AutoDream (记忆整合) —— "睡眠整理"

定期运行的后台代理，对持久记忆做整理、合并、去重、清理。灵感来自人类睡眠时整理白天记忆的过程。

```
        AutoDream — Memory Consolidation
        ┌──────────────────────────────────┐
        │     3 Gates (all must pass):      │
        │   ┌──────┐  ┌──────┐  ┌──────┐   │
        │   │ >24h │  │ >5   │  │ Lock │   │
        │   │since │  │sess- │  │acqu- │   │
        │   │last  │  │ions  │  │ired  │   │
        │   └──────┘  └──────┘  └──────┘   │
        │                                   │
        │        Start Dreaming             │
        │                                   │
        │  4 Phases:                        │
        │  Orient → Gather → Consolidate    │
        │  → Prune                          │
        │                                   │
        │  Read-only code access,           │
        │  write-only to memory directory   │
        │                                   │
        │  "You are performing a dream —    │
        │   a reflective pass over your     │
        │   memory files."                  │
        └──────────────────────────────────┘
```

### 触发条件 (三个门全部通过才执行):

1. 距上次整合 > 24 小时 (最先检查，成本最低——一次 stat 调用)
2. 距上次整合 > 5 个会话
3. 获取 consolidation lock (PID 文件锁，防并发; 超过 1 小时且 PID 已死则可回收)

### 执行 4 阶段:

- **Orient**: 读取 MEMORY.md 索引，浏览现有主题文件，了解当前状态
- **Gather**: 扫描每日日志和会话记录 (JSONL) ，用 grep 范围搜索，不穷举
- **Consolidate**: 新信息合并到现有主题文件，修复矛盾，相对日期转绝对日期
- **Prune**: 保持 MEMORY.md < 200 行，删除过时条目，精简冗余描述

System prompt: "You are performing a dream —a reflective pass over your memory files. Synthesize what you've learned recently into durable, well-organized memories so that future sessions can orient quickly."

整合期间只读访问——可以看代码文件，但只能写记忆目录。由 Fork Agent 执行，UI 底栏显示 "dreaming" 标签。

---

### 三者对比

| 维度 | extractMemories | Session Memory | AutoDream |
|------|----------------|----------------|-----------|
| **功能** | 从对话中提取新记忆写入持久存储 | 维护当前会话的结构化工作笔记 | 定期整理已有的持久记忆 |
| **写入目标** | `~/.claude/memory/*.md` + MEMORY.md 索引 | Session 状态文件 | `~/.claude/memory/*.md` + MEMORY.md 索引 |
| **生命周期** | 长期 (跨会话) | 仅当前会话 | 长期 (跨会话) |
| **操作性质** | 新增/更新记忆条目 | 维护/覆盖一份工作纪要 | 合并/去重/删除/精简已有条目 |
| **触发频率** | 每轮查询结束后 (主 Agent 自己没写记忆时) | token 超 10K 后启动，每 5K tokens 更新 | >24h 且 >5 个会话 且 获取文件锁 |
| **频率** | 高 (每轮或每N轮) | 中 (按 token 增量) | 极低 (按天) |
| **信息来源** | 最近 N 条对话消息 | 整个当前对话 | 已有记忆文件 + 会话日志 |
| **写什么** | user/feedback/project/reference 四类持久事实 | 任务描述、进度、涉及文件、错误、经验 | 合并同类、删除过时、转绝对日期 |
| **下游用途** | 持久记忆加载时当候选 | 1. resume 恢复会话 2. 压缩时当现成摘要 | 保持记忆库整洁、<200 行 |
| **权限范围** | 只读代码 + 只写 memory 目录 | 读写 Session 状态 | 只读代码 + 只写 memory 目录 |
| **执行方式** | Fork Agent 后台执行 | Fork Agent 后台执行 | Fork Agent 后台执行 (UI 显示 "dreaming") |
| **类比** | 上课记笔记 | 写会议纪要 | 周末整理笔记本 |

---

## 4.4.2.5 Claude Code vs OpenClaw: 记忆设计思路对比

两者解决的是同一个问题——有限上下文窗口下如何让 Agent 不失忆，但设计思路有几处根本性的分歧。

### 检索路线之争: Agentic Search vs RAG

这是两者最大的分歧。Claude Code 明确拒绝 RAG，Boris 透露早期试过向量数据库 + embedding 方案，但发现代码修改后索引不同步、权限管控复杂，最终结论是让 Agent 自己调 glob + grep 实时搜索 (agentic search) 全面优于 RAG。OpenClaw 则走了完全相反的路——用 SQLite 建本地索引，向量语义匹配 + BM25 关键词匹配加权合并做混合检索。

本质上这是 "每次多花点 token 换实时准确" vs "建索引省 token 但要维护一致性" 的取舍。对 coding agent 来说，代码文件变化极频繁，索引的一致性成本很高，所以 Claude Code 选择了不建索引; 而 OpenClaw 面向更通用的 Agent 场景 (聊天、知识管理) ，记忆文件相对稳定，RAG 的收益更明显。

### 压缩策略: 梯度递进 vs 一步到位

Claude Code 设计了 5 层压缩流水线，从温和到激进逐层升级——先清理工具输出碎片、再考虑缓存优化、然后才做 LLM 摘要、最后硬拒绝和兜底。大部分情况下前两层 (不需要 LLM 调用) 就能把 token 压下来，真正的 LLM 压缩是中低频事件。这是一种用工程复杂度换成本效率的思路。

OpenClaw 只有一级压缩，但流程更精细: 划分保留区(最近 ~20K tokens 原封不动) 、压缩区分块逐块总结、多块摘要再合并成一个连贯总结。每次触发都需要 LLM 调用，成本更高，但实现简单、行为可预测。

### 记忆写入: 自动化 vs 显式写入

Claude Code 做了重度的写入自动化——三个后台 Agent 分工: extractMemories 每轮结束后自动提取持久记忆，Session Memory 持续维护会话笔记，AutoDream 定期整合去重。用户不需要主动"存档"，系统自动帮你把重要信息沉淀下来。

OpenClaw 更依赖显式写入。它的文档反复强调一个常见失误: 用户说过的事情不会自动保存到文件，不写就会丢失。因此，OpenClaw 在压缩前有一个 Pre-Compaction Memory Flush——静默提醒 Agent 把重要内容写入记忆文件。但这本质上是"事到临头才保存"，而 Claude Code 是"一直在保存"。

### 记忆加载: Push vs Pull

Claude Code 是系统层面硬编码的 push 模式——每轮对话前，系统必定用 Sonnet (轻量小模型) 扫一遍所有记忆文件的标题和描述，挑出最相关的几条，直接塞进上下文。主模型开始思考时，相关记忆已经摆在眼前了，不需要它主动去查。比喻: 秘书在你开会前，把相关资料提前放在桌上。代价是每轮都要占固定的上下文空间，不管用不用得上。

OpenClaw 是 pull 模式——系统不会提前塞记忆，而是把 memory_search 作为一个普通工具暴露给模型。模型在推理过程中自己判断"这轮要不要查记忆"，跟它决定要不要调 bash、要不要读文件是一样的逻辑。更灵活省空间，但风险在于模型不知道自己不知道什么——比如你上周说过"我讨厌用 forEach"，这条记忆存在文件里了，但今天聊到循环时模型可能根本没想到要去搜"用户的代码风格偏好"，就直接写了 forEach。Push 模式下这条记忆会被小模型提前捞出来，就不会犯这个错。

### 设计哲学总结

Claude Code 走的是工程优化路线: 梯度压缩能省则省、后台 Agent 自动写入、小模型做筛选降低成本——处处体现"怎么用更少的 token 做更多的事"。OpenClaw 走的是架构清晰路线: 四层对标计算机存储层次 (寄存器→缓存→内存→搜索引擎) ，每层职责明确，系统更易理解和调试。面试时两者结合讲，既能展示工程优化思维，又能体现架构设计能力。
