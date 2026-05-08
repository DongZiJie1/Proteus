# Proteus — 项目规格说明书

## 1. 愿景

Proteus 是一个通用 AI Agent，能像人一样操作电脑——读屏幕、点鼠标、敲键盘、操控浏览器，并通过 CLI 或消息渠道（微信、飞书）与用户交互。

一句话定位：**一个住在你电脑里的 AI 助手，你用消息跟它说话，它帮你操作电脑完成任务。**

## 2. 目标用户

| 用户 | 场景 |
|------|------|
| 开发者 | 通过 CLI 让 Agent 执行代码、跑测试、操控浏览器做自动化测试 |
| 运营/办公人员 | 通过微信/飞书发消息，让 Agent 代操作桌面软件（填表、导数据、发通知） |
| 个人用户 | 日常任务委派——订票、比价、整理文件、网页信息采集 |

## 3. 核心概念

```
┌─────────────────────────────────────────────────────┐
│                    Proteus 架构                      │
│                                                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐          │
│  │   CLI    │  │  微信    │  │  飞书    │  ← 渠道层 │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘          │
│       │             │             │                 │
│       └─────────────┼─────────────┘                 │
│                     │                               │
│              ┌──────▼──────┐                        │
│              │  Agent Loop │  ← 核心循环             │
│              │  (思考+决策) │                        │
│              └──────┬──────┘                        │
│                     │                               │
│         ┌───────────┼───────────┐                   │
│         ▼           ▼           ▼                   │
│   ┌──────────┐ ┌──────────┐ ┌──────────┐           │
│   │ LLM 层  │ │ 工具层   │ │ 记忆层   │           │
│   └──────────┘ └──────────┘ └──────────┘           │
│                     │                               │
│    ┌────────┬───────┼───────┬────────┐              │
│    ▼        ▼       ▼       ▼        ▼              │
│  电脑    浏览器   文件    Shell    网页              │
│  操控    操控    操作    命令     搜索              │
└─────────────────────────────────────────────────────┘
```

### 3.1 Agent Loop（核心循环）

Agent 的行为模型：

```
while not done and step < max_steps:
    1. 构建 prompt = system_prompt + conversation_history + tool_results
    2. 调用 LLM → 得到 response（可能包含 tool_calls）
    3. 如果 response 是纯文本 → 返回给用户，done
    4. 如果 response 包含 tool_calls:
       a. 逐个执行工具（异步，带超时）
       b. 将工具结果追加到 conversation_history
       c. 回到步骤 1
    5. 如果连续 3 次工具调用无实质进展 → 提示用户介入
```

### 3.2 工具系统

工具是 Agent 的"手脚"。每个工具包含以下要素：

| 字段 | 类型 | 说明 |
|------|------|------|
| name | string | 工具名，LLM 用来引用 |
| description | string | 工具描述，告诉 LLM 这个工具能干嘛 |
| parameters | JSON Schema | 定义参数格式 |
| execute() | async method | 异步执行方法，接收参数返回 ToolResult |

工具通过装饰器注册到全局注册中心，注册时自动生成 JSON Schema 供 LLM function calling 使用。

### 3.3 渠道系统

渠道是 Agent 的"嘴和耳朵"。每个渠道需实现四个方法：

| 方法 | 说明 |
|------|------|
| start() | 启动渠道监听 |
| stop() | 停止渠道 |
| receive() | 接收用户消息（异步迭代器） |
| send() | 发送消息给用户 |

## 4. 功能规格

### 4.1 CLI 渠道

**核心模式：连续对话**

CLI 的主要使用方式是连续对话。Agent 保持上下文，用户可以多轮交互，逐步推进任务。Agent 在执行过程中会主动向用户反馈进度、询问确认、请求补充信息。

```bash
$ proteus
> 帮我打开浏览器，搜索 Python 3.12 的新特性
（Agent 打开浏览器，搜索，返回结果摘要）
> 打开第一条结果看看详情
（Agent 点击链接，抓取页面内容）
> 把关键内容整理成 Markdown 保存到桌面
（Agent 整理内容，写入文件）
> 再帮我搜一下和 3.11 的对比
（Agent 继续搜索，结合上下文给出对比）
```

对话过程中 Agent 可以：
- 多步操作后主动汇报中间结果
- 遇到歧义时暂停并询问用户
- 任务复杂时先给出计划再执行
- 执行失败时说明原因并建议替代方案
- 用户随时用 `Ctrl+C` 中断当前任务，对话不中断

**单次模式（快捷方式）：**

```bash
# 传入任务描述，完成后退出（适合脚本和快捷操作）
proteus "帮我打开浏览器搜索今天的天气"
```

**指定渠道启动：**

```bash
proteus --channel wechat
proteus --channel feishu
```

**CLI 特性：**
- 流式输出（LLM token 边生成边显示）
- Markdown 渲染（终端富文本）
- 工具调用过程可视化（显示 Agent 正在执行什么操作）
- Vim 模式支持（可选）

**斜杠命令：**

| 命令 | 说明 |
|------|------|
| `/help` | 显示帮助信息和可用命令列表 |
| `/clear` | 清空当前对话历史 |
| `/compact` | 压缩对话历史（摘要旧对话，释放上下文窗口） |
| `/history` | 查看对话历史列表 |
| `/model` | 切换 LLM 模型（如 claude-sonnet / gpt-4o） |
| `/config` | 查看或修改配置项 |
| `/permissions` | 查看和管理工具权限（auto/confirm/deny） |
| `/memory` | 管理长期记忆（查看/编辑/删除记忆条目） |
| `/cost` | 显示当前会话的 token 用量和费用统计 |
| `/status` | 显示当前状态（模型、渠道、工具开关等） |
| `/doctor` | 系统健康检查（依赖、API 连通性、工具可用性） |
| `/init` | 初始化项目配置文件 |
| `/login` | 登录 LLM Provider（配置 API Key） |
| `/logout` | 登出当前 Provider |
| `/review` | 代码审查（对当前 diff 或指定文件进行审查） |
| `/terminal-setup` | 终端集成配置（快捷键、shell 集成） |
| `/theme` | 切换终端主题（深色/浅色/自定义） |
| `/mcp` | 管理 MCP 服务器连接 |
| `/bug` | 报告 Bug（自动收集上下文信息） |
| `/insights` | 分析历史会话，提取使用模式、高频话题、工具偏好、token 消耗趋势等洞察 |
| `/quit` | 退出 Proteus |
| `/exit` | 退出 Proteus（同 /quit） |

**快捷键：**

| 快捷键 | 说明 |
|--------|------|
| `Ctrl+C` | 中断当前生成/任务 |
| `Ctrl+D` | 退出（等同 /quit） |
| `Escape` | 取消当前输入 |
| `Up/Down` | 浏览输入历史 |
| `Tab` | 命令/路径自动补全 |
| `Ctrl+L` | 清屏 |

**启动参数：**

| 参数 | 说明 |
|------|------|
| `proteus "任务描述"` | 单次任务模式，完成后退出 |
| `proteus` | 交互式对话模式 |
| `proteus --channel wechat` | 指定渠道启动 |
| `proteus --channel feishu` | 指定渠道启动 |
| `proteus --model gpt-4o` | 指定 LLM 模型 |
| `proteus --config path/to/config.yaml` | 指定配置文件 |
| `proteus --no-browser` | 禁用浏览器工具 |
| `proteus --no-computer` | 禁用电脑操控工具 |
| `proteus --auto-confirm` | 跳过所有确认（危险） |
| `proteus --verbose` | 详细日志输出 |
| `proteus --version` | 显示版本号 |

### 4.2 电脑操控工具

| 工具名 | 功能 | 关键实现 |
|--------|------|----------|
| `screenshot` | 截取屏幕（全屏/区域） | Pillow + pyautogui |
| `mouse_click` | 点击指定坐标 | pyautogui，支持左/右/双击 |
| `mouse_move` | 移动鼠标 | pyautogui |
| `mouse_drag` | 拖拽操作 | pyautogui |
| `type_text` | 输入文字 | pyautogui，支持中英文 |
| `press_key` | 按键（Enter/Ctrl+C 等） | pyautogui |
| `scroll` | 滚动页面 | pyautogui |
| `get_screen_text` | OCR 识别屏幕文字 | 内置轻量 OCR 或调用 API |
| `list_windows` | 列出当前窗口 | pyautogui / subprocess |
| `focus_window` | 切换到指定窗口 | pyautogui |

**操控流程：**

```
截图 → LLM 分析截图内容 → 决定下一步操作 → 执行操作 → 再截图确认 → 循环
```

Agent 必须遵循"看一步做一步"原则：每执行一个操作后截图确认状态，再决定下一步。

### 4.3 浏览器操控工具

| 工具名 | 功能 | 关键实现 |
|--------|------|----------|
| `browser_launch` | 启动浏览器 | Playwright |
| `browser_goto` | 导航到 URL | Playwright |
| `browser_click` | 点击页面元素 | 优先 accessibility tree 定位 |
| `browser_type` | 在输入框输入文字 | Playwright |
| `browser_select` | 选择下拉选项 | Playwright |
| `browser_screenshot` | 截取页面截图 | Playwright |
| `browser_get_text` | 获取页面文本内容 | Playwright |
| `browser_get_links` | 获取页面链接 | Playwright |
| `browser_execute_js` | 执行 JavaScript | Playwright |
| `browser_tabs` | 管理标签页 | Playwright |
| `browser_scroll` | 滚动页面 | Playwright |

**元素定位策略（优先级）：**

1. Accessibility tree（`aria-label`、`role`、`name`）— 最稳定
2. CSS 选择器
3. XPath
4. 坐标定位（兜底）

### 4.4 文件与 Shell 工具

| 工具名 | 功能 |
|--------|------|
| `read_file` | 读取文件内容 |
| `write_file` | 写入文件 |
| `edit_file` | 编辑文件（查找替换） |
| `list_files` | 列出目录内容 |
| `search_files` | 按文件名/内容搜索 |
| `run_command` | 执行 Shell 命令 |

**Shell 命令约束：**
- 默认超时 30s，可配置
- 需要用户确认的命令：`rm -rf`、`sudo`、涉及系统目录的写操作
- 输出截断：超过 10KB 的输出自动截断，提示 Agent 用更精确的命令

### 4.5 网页工具

| 工具名 | 功能 |
|--------|------|
| `web_search` | 搜索引擎搜索 |
| `web_fetch` | 抓取网页内容（转 Markdown） |
| `web_extract` | 从网页提取结构化数据 |

### 4.6 记忆系统

记忆系统统一管理三个层级，不是独立子系统而是同一套体系的不同深度：

**工作记忆（上下文窗口）：**
- 系统提示词 + MEMORY.md 索引（始终加载）
- 用户指令 + 近期对话历史 + 工具调用结果
- 容量有限，需要压缩管理：工具结果摘要化 → 对话历史摘要化 → 记忆文件卸载 → 截断

**短期记忆（会话历史）：**
- 本次会话的完整对话记录 + 工具调用记录
- 存在内存中，会话结束后归档到本地 JSONL 文件
- 每条消息一行 JSON，包含角色、内容、工具调用、时间戳等完整信息
- 存储路径：`~/.proteus/projects/{project_hash}/conversations/YYYY-MM-DD_HHMMSS.jsonl`
- 支持按会话列表、按时间范围检索、回放任意历史会话

**长期记忆（持久记忆文件）：**
- 四种类型：user（用户画像）、feedback（行为反馈）、project（项目动态）、reference（外部资源）
- 以 Markdown 文件存储，MEMORY.md 为索引（每次会话自动加载）
- 通过 autodream 机制从上下文中自动沉淀

**Autodream（自动沉淀）：**
- 每 N 轮对话或会话结束时，异步触发独立 LLM 调用
- 回顾近期对话，提取有价值信息写入长期记忆
- 不阻塞主流程，后台执行

### 4.7 微信渠道

**接入方式：** 基于微信机器人框架（如 wechaty / itchat 替代方案），或通过企业微信 API。

**支持的消息类型：**
- 文本消息（主要）
- 图片消息（Agent 可以发送截图给用户确认）
- 文件消息（Agent 可以发送生成的文件）

**约束：**
- 微信个人号接入不稳定，优先支持企业微信
- 消息长度限制：单条不超过 2000 字符，超过自动分段
- 不支持主动推送（仅响应用户消息）

### 4.8 飞书渠道

**接入方式：** 飞书开放平台 Bot API。

**支持的消息类型：**
- 文本、富文本、Markdown
- 图片、文件
- 交互式卡片（用于确认操作、展示结果）

**优势：**
- 飞书 Bot API 稳定，文档完善
- 支持卡片消息，交互体验好
- 支持群聊 @机器人 触发

## 5. 数据模型

### 消息（Message）

| 字段 | 类型 | 说明 |
|------|------|------|
| id | string | 消息唯一 ID |
| role | enum | user / assistant / system / tool |
| content | string | 文本内容 |
| attachments | list | 附件列表（图片、文件、音频） |
| tool_calls | list | 工具调用请求 |
| tool_results | list | 工具执行结果 |
| timestamp | datetime | 消息时间戳 |
| channel | string | 来源渠道 |
| metadata | dict | 渠道特有元数据 |

### 附件（Attachment）

| 字段 | 类型 | 说明 |
|------|------|------|
| type | enum | image / file / audio |
| content | bytes | 二进制内容（与 url 二选一） |
| url | string | 附件 URL（与 content 二选一） |
| mime_type | string | MIME 类型 |
| filename | string | 文件名 |

### 工具调用（ToolCall）

| 字段 | 类型 | 说明 |
|------|------|------|
| id | string | 调用唯一 ID |
| name | string | 工具名 |
| arguments | dict | 参数字典 |

### 工具结果（ToolResult）

| 字段 | 类型 | 说明 |
|------|------|------|
| tool_call_id | string | 对应的工具调用 ID |
| content | string | 文本结果 |
| images | list | 图片结果（如截图） |
| is_error | bool | 是否错误 |
| error_message | string | 错误信息 |

### 会话（Session）

| 字段 | 类型 | 说明 |
|------|------|------|
| id | string | 会话唯一 ID |
| channel | string | 来源渠道 |
| messages | list | 消息列表 |
| created_at | datetime | 创建时间 |
| metadata | dict | 元数据 |

## 6. LLM 层设计

### 6.1 多 Provider 支持

统一的 LLM Provider 接口，包含两个核心方法：

| 方法 | 说明 |
|------|------|
| chat() | 同步对话，接收消息列表和工具 schema，返回 LLM 响应 |
| chat_stream() | 流式对话，返回 token 异步迭代器 |

支持的 Provider：
- Anthropic（Claude）— 默认
- OpenAI（GPT-4o）
- 其他 OpenAI 兼容 API（通过 base_url 配置）

### 6.2 Function Calling

所有工具自动转换为 LLM 的 function calling schema。Agent Loop 解析 LLM 响应中的 tool_use 块，执行对应工具。

### 6.3 上下文窗口管理

- 对话历史超过上下文窗口时，自动压缩（摘要旧对话）
- 截图等大附件在历史中替换为文本描述
- 可配置保留最近 N 轮对话

## 7. 安全设计

### 7.1 权限分级

```yaml
auto:          # 自动执行，无需确认
  - screenshot
  - list_files
  - read_file
  - web_search
  - web_fetch

confirm:       # 执行前需用户确认
  - write_file
  - edit_file
  - run_command（非白名单命令）
  - browser_goto（非白名单域名）
  - mouse_click（涉及系统 UI）

deny:          # 默认禁止
  - run_command: "rm -rf /"
  - 访问系统关键目录（/etc, /System）
```

### 7.2 确认机制

CLI 渠道：终端内交互确认（y/n）
消息渠道：发送确认卡片，用户点击按钮确认

### 7.3 操作审计

所有工具调用记录到本地日志，包含：
- 时间戳
- 工具名 + 参数
- 执行结果
- 截图（如果是操控类工具）

## 8. Hook 机制

Hook 是 Proteus 的事件钩子系统，允许用户在 Agent 生命周期的关键节点注入自定义逻辑（脚本、通知、日志、审批流等）。

### 8.1 工作原理

每个 Hook 绑定一个事件点，当该事件触发时，Proteus 执行用户配置的 Shell 命令或脚本。Hook 可以：

- **检查** — 审查输入/输出，决定是否放行
- **修改** — 在执行前修改参数或结果
- **通知** — 发送消息到外部系统（Slack、钉钉、日志服务等）
- **拦截** — 阻止操作执行（返回非零退出码则拦截）

### 8.2 事件点

| 事件 | 触发时机 | 输入数据 | 能否拦截 |
|------|----------|----------|----------|
| `session_start` | 会话创建时 | 渠道类型、用户信息、配置 | 否 |
| `session_end` | 会话结束时 | 会话 ID、持续时间、消息数、工具调用统计 | 否 |
| `user_message` | 收到用户消息时 | 消息内容、来源渠道、附件列表 | 是（可拦截消息不让 Agent 处理） |
| `agent_message` | Agent 发送回复时 | 回复内容、token 用量 | 是（可修改或拦截回复） |
| `pre_llm_call` | 调用 LLM 之前 | 完整 prompt、工具列表、模型名 | 是（可修改 prompt 或拦截调用） |
| `post_llm_call` | LLM 响应返回后 | 响应内容、token 用量、耗时 | 是（可修改响应） |
| `pre_tool_use` | 工具执行之前 | 工具名、参数、权限级别 | 是（可修改参数或拦截执行） |
| `post_tool_use` | 工具执行之后 | 工具名、参数、执行结果、耗时 | 是（可修改结果） |
| `tool_error` | 工具执行失败时 | 工具名、参数、错误信息、重试次数 | 是（可决定是否重试） |
| `task_start` | Agent 开始处理任务时 | 任务描述、来源渠道 | 否 |
| `task_complete` | Agent 完成任务时 | 任务描述、总步数、总耗时、token 用量 | 否 |
| `permission_request` | 请求用户确认时 | 工具名、参数、权限级别 | 是（可自动批准或拒绝） |
| `pre_compact` | 上下文压缩之前 | 当前对话长度、token 用量 | 是（可跳过压缩） |
| `post_compact` | 上下文压缩之后 | 压缩前后的 token 对比 | 否 |
| `error` | Agent 未捕获异常时 | 异常类型、堆栈、当前上下文 | 否 |

### 8.3 配置方式

在配置文件中定义 Hook：

```yaml
hooks:
  pre_tool_use:
    - command: "python ~/.proteus/hooks/audit.py"
      tools: ["run_command", "write_file"]  # 只对指定工具触发，空则全部
      timeout: 5

  post_tool_use:
    - command: "python ~/.proteus/hooks/notify.py"
      tools: ["screenshot"]

  permission_request:
    - command: "python ~/.proteus/hooks/auto_approve.py"
      timeout: 10

  session_end:
    - command: "python ~/.proteus/hooks/session_report.py"
```

### 8.4 Hook 执行规则

- Hook 按配置顺序依次执行
- 可拦截的 Hook：退出码 0 = 放行，非 0 = 拦截
- 拦截类 Hook 通过 stdout 输出修改后的数据（JSON 格式）
- Hook 超时后自动跳过，不阻塞主流程
- Hook 执行失败不影响 Agent 主流程（仅记录日志）
- 环境变量 `PROTEUS_HOOK_EVENT` 传入当前事件名
- 环境变量 `PROTEUS_HOOK_DATA` 传入事件数据（JSON）

## 9. 配置

配置文件 `~/.proteus/config.yaml` 或环境变量：

```yaml
# LLM 配置
llm:
  provider: anthropic          # anthropic / openai / custom
  model: claude-sonnet-4-20250514
  api_key: sk-xxx              # 或通过环境变量 PROTEUS_LLM_API_KEY
  base_url: null               # 自定义 API 地址
  max_tokens: 4096

# Agent 配置
agent:
  max_steps: 50                # 单次任务最大步骤数
  max_tool_timeout: 30         # 工具超时秒数
  context_window: 128000       # 上下文窗口大小
  auto_confirm: false          # 是否跳过确认（危险）

# 渠道配置
channels:
  cli:
    enabled: true
    stream: true               # 流式输出
  wechat:
    enabled: false
    bot_token: null
  feishu:
    enabled: false
    app_id: null
    app_secret: null

# 工具开关
tools:
  computer_control: true
  browser: true
  shell: true
  file_ops: true
  web: true

# 安全配置
security:
  confirm_level: confirm       # auto / confirm / deny
  allowed_commands: []         # Shell 命令白名单
  blocked_commands:            # Shell 命令黑名单
    - "rm -rf /"
    - "sudo rm"
  allowed_domains: []          # 浏览器域名白名单

# Hook 配置
hooks:
  session_start: []
  session_end: []
  user_message: []
  agent_message: []
  pre_llm_call: []
  post_llm_call: []
  pre_tool_use: []
  post_tool_use: []
  tool_error: []
  task_start: []
  task_complete: []
  permission_request: []
  pre_compact: []
  post_compact: []
  error: []
```

## 10. 非目标（明确不做）

- **不做通用聊天机器人** — Proteus 是任务导向的，不做闲聊
- **不做 SaaS 平台** — 本地部署，不托管用户数据
- **不做 LangChain 封装** — 自研轻量 Agent 框架
- **不做训练/微调** — 只做推理，不训练模型
- **不做 GUI 客户端** — 初期只做 CLI + 消息渠道

## 11. 性能指标

| 指标 | 目标 |
|------|------|
| CLI 首次响应 | < 2s（不含 LLM 推理） |
| 工具执行延迟 | < 1s（除浏览器/电脑操控） |
| 截图到分析完成 | < 5s |
| 单次任务最大耗时 | 可配置，默认 5min |
| 内存占用 | < 200MB（不含浏览器） |
| 并发会话 | 支持多渠道同时响应 |
