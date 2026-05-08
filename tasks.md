# Proteus — 原子需求任务清单

> 从 spec.md 拆解的所有原子任务，`[ ]` 未完成，`[x]` 已完成。

---

## A. 项目基础设施

- [ ] A01 初始化 Python 项目（pyproject.toml、uv 配置）
- [ ] A02 搭建目录结构（src/proteus/ 下各子模块）
- [ ] A03 配置 ruff（lint + format 规则）
- [ ] A04 配置 mypy strict 模式
- [ ] A05 配置 pytest（测试目录、conftest、标记）
- [ ] A06 配置 loguru 日志（格式、级别、输出）
- [ ] A07 编写 .gitignore
- [ ] A08 编写 .env.example（所有配置项说明）

---

## B. 配置管理

- [ ] B01 定义配置模型（pydantic-settings，所有配置项）
- [ ] B02 实现环境变量读取（PROTEUS_ 前缀）
- [ ] B03 实现配置文件读取（~/.proteus/config.yaml）
- [ ] B04 实现配置文件生成（/init 命令）
- [ ] B05 实现配置优先级（环境变量 > 配置文件 > 默认值）

---

## C. 数据模型

- [ ] C01 定义 Message 模型
- [ ] C02 定义 Attachment 模型
- [ ] C03 定义 ToolCall 模型
- [ ] C04 定义 ToolResult 模型
- [ ] C05 定义 Session 模型
- [ ] C06 定义 LLMResponse 模型（包含文本 + tool_calls）
- [ ] C07 定义 ToolSchema 模型（工具的 JSON Schema 描述）

---

## D. LLM 层

- [ ] D01 定义 LLMProvider 抽象接口（chat / chat_stream）
- [ ] D02 实现 Anthropic Provider（Claude 系列模型）
- [ ] D03 实现 OpenAI Provider（GPT 系列模型）
- [ ] D04 实现 OpenAI 兼容 Provider（自定义 base_url）
- [ ] D05 实现 Provider 工厂（根据配置创建对应 Provider）
- [ ] D06 实现流式输出（逐 token 异步迭代）
- [ ] D07 实现 Function Calling（工具 schema 注入 + 响应解析）
- [ ] D08 实现 LLM 调用重试（指数退避 + 最大重试次数）
- [ ] D09 实现 Provider 降级（主 Provider 失败自动切换备用）
- [ ] D10 实现 token 用量统计（输入/输出/总计）

---

## E. 工具系统

- [ ] E01 定义 ToolBase 抽象基类（name/description/parameters/execute）
- [ ] E02 实现工具注册中心（装饰器注册 + 全局查找）
- [ ] E03 实现工具 JSON Schema 自动生成（供 LLM function calling）
- [ ] E04 实现工具执行器（异步执行 + 超时控制）
- [ ] E05 实现工具结果封装（文本 + 图片 + 错误）

---

## F. 文件与 Shell 工具

- [ ] F01 实现 read_file 工具
- [ ] F02 实现 write_file 工具
- [ ] F03 实现 edit_file 工具（查找替换）
- [ ] F04 实现 list_files 工具
- [ ] F05 实现 search_files 工具（按文件名/内容搜索）
- [ ] F06 实现 run_command 工具（Shell 命令执行）
- [ ] F07 实现 run_command 超时控制（默认 30s）
- [ ] F08 实现 run_command 输出截断（超过 10KB 截断）

---

## G. 网页工具

- [ ] G01 实现 web_search 工具（搜索引擎搜索）
- [ ] G02 实现 web_fetch 工具（抓取网页内容转 Markdown）
- [ ] G03 实现 web_extract 工具（从网页提取结构化数据）

---

## H. 浏览器操控工具

- [ ] H01 集成 Playwright（启动/关闭浏览器实例）
- [ ] H02 实现 browser_launch 工具
- [ ] H03 实现 browser_goto 工具（导航到 URL）
- [ ] H04 实现 browser_click 工具（点击页面元素）
- [ ] H05 实现 browser_type 工具（输入框输入文字）
- [ ] H06 实现 browser_select 工具（下拉选项选择）
- [ ] H07 实现 browser_screenshot 工具（截取页面截图）
- [ ] H08 实现 browser_get_text 工具（获取页面文本）
- [ ] H09 实现 browser_get_links 工具（获取页面链接）
- [ ] H10 实现 browser_execute_js 工具（执行 JavaScript）
- [ ] H11 实现 browser_tabs 工具（标签页管理）
- [ ] H12 实现 browser_scroll 工具（滚动页面）
- [ ] H13 实现元素定位策略（accessibility tree 优先）
- [ ] H14 实现元素定位降级（CSS → XPath → 坐标）

---

## I. 电脑操控工具

- [ ] I01 实现 screenshot 工具（全屏截图）
- [ ] I02 实现 screenshot 区域截图（指定 [x, y, w, h]）
- [ ] I03 实现 mouse_click 工具（左/右/双击）
- [ ] I04 实现 mouse_move 工具
- [ ] I05 实现 mouse_drag 工具
- [ ] I06 实现 type_text 工具（支持中英文输入）
- [ ] I07 实现 press_key 工具（Enter/Ctrl+C 等按键）
- [ ] I08 实现 scroll 工具
- [ ] I09 实现 get_screen_text 工具（OCR 识别屏幕文字）
- [ ] I10 实现 list_windows 工具（列出当前窗口）
- [ ] I11 实现 focus_window 工具（切换到指定窗口）

---

## J. Agent Loop

- [ ] J01 实现 Agent Loop 主循环（接收输入 → LLM 推理 → 工具调用 → 返回）
- [ ] J02 实现对话上下文维护（conversation_history 管理）
- [ ] J03 实现工具调用调度（解析 LLM tool_calls，逐个执行）
- [ ] J04 实现最大迭代次数限制（防止死循环，默认 50 步）
- [ ] J05 实现无进展检测（连续 3 次无实质进展提示用户）
- [ ] J06 实现 LLM 交互日志记录（完整 prompt + response）
- [ ] J07 实现流式 Agent Loop（边推理边输出 token）
- [ ] J08 实现任务中断（用户 Ctrl+C 中断当前任务，不中断对话）

---

## K. 安全系统

- [ ] K01 实现权限分级模型（auto/confirm/deny 三级）
- [ ] K02 实现工具权限配置（每个工具对应权限级别）
- [ ] K03 实现 CLI 确认机制（终端 y/n 交互确认）
- [ ] K04 实现消息渠道确认机制（确认卡片按钮）
- [ ] K05 实现 Shell 命令黑名单（rm -rf /、sudo rm 等）
- [ ] K06 实现 Shell 命令白名单（可配置的允许命令列表）
- [ ] K07 实现浏览器域名白名单
- [ ] K08 实现操作审计日志（时间戳 + 工具 + 参数 + 结果 + 截图）

---

## L. Hook 机制

- [ ] L01 定义 Hook 事件模型（事件名、输入数据、拦截标志）
- [ ] L02 实现 Hook 注册与管理（配置文件加载）
- [ ] L03 实现 Hook 执行器（Shell 命令/脚本调用、超时控制）
- [ ] L04 实现 Hook 拦截机制（退出码判断 + stdout 数据回写）
- [ ] L05 实现 session_start 事件触发
- [ ] L06 实现 session_end 事件触发
- [ ] L07 实现 user_message 事件触发（可拦截）
- [ ] L08 实现 agent_message 事件触发（可拦截/修改）
- [ ] L09 实现 pre_llm_call 事件触发（可修改 prompt / 拦截）
- [ ] L10 实现 post_llm_call 事件触发（可修改响应）
- [ ] L11 实现 pre_tool_use 事件触发（可修改参数 / 拦截执行）
- [ ] L12 实现 post_tool_use 事件触发（可修改结果）
- [ ] L13 实现 tool_error 事件触发（可决定是否重试）
- [ ] L14 实现 task_start 事件触发
- [ ] L15 实现 task_complete 事件触发
- [ ] L16 实现 permission_request 事件触发（可自动批准/拒绝）
- [ ] L17 实现 pre_compact 事件触发（可跳过压缩）
- [ ] L18 实现 post_compact 事件触发
- [ ] L19 实现 error 事件触发（未捕获异常）
- [ ] L20 实现 Hook 环境变量注入（PROTEUS_HOOK_EVENT / PROTEUS_HOOK_DATA）

---

## M. CLI 渠道

- [ ] M01 实现 CLI 渠道基类（start/stop/receive/send）
- [ ] M02 实现交互式对话模式（连续多轮对话）
- [ ] M03 实现单次任务模式（传入任务描述，完成后退出）
- [ ] M04 实现流式输出（逐 token 边生成边显示）
- [ ] M05 实现终端 Markdown 渲染
- [ ] M06 实现工具调用过程可视化（显示 Agent 正在执行什么）
- [ ] M07 实现斜杠命令 /help
- [ ] M08 实现斜杠命令 /clear
- [ ] M09 实现斜杠命令 /compact
- [ ] M10 实现斜杠命令 /history
- [ ] M11 实现斜杠命令 /model（切换 LLM 模型）
- [ ] M12 实现斜杠命令 /config（查看/修改配置）
- [ ] M13 实现斜杠命令 /permissions（管理工具权限）
- [ ] M14 实现斜杠命令 /memory（管理长期记忆）
- [ ] M15 实现斜杠命令 /cost（token 用量和费用统计）
- [ ] M16 实现斜杠命令 /status（显示当前状态）
- [ ] M17 实现斜杠命令 /doctor（系统健康检查）
- [ ] M18 实现斜杠命令 /init（初始化配置文件）
- [ ] M19 实现斜杠命令 /login（配置 API Key）
- [ ] M20 实现斜杠命令 /logout
- [ ] M21 实现斜杠命令 /review（代码审查）
- [ ] M22 实现斜杠命令 /terminal-setup
- [ ] M23 实现斜杠命令 /theme（切换主题）
- [ ] M24 实现斜杠命令 /mcp（管理 MCP 服务器）
- [ ] M25 实现斜杠命令 /bug（报告 Bug）
- [ ] M26 实现斜杠命令 /quit 和 /exit
- [ ] M27 实现快捷键（Ctrl+C/Ctrl+D/Escape/Up/Down/Tab/Ctrl+L）
- [ ] M28 实现输入历史浏览（上下箭头）
- [ ] M29 实现 Tab 自动补全（命令/路径）
- [ ] M30 实现 Vim 模式（可选）
- [ ] M31 实现启动参数解析（--channel/--model/--config/--no-browser 等）
- [ ] M32 实现 /hook 命令（查看/管理已注册 Hook）
- [ ] M33 实现 /insights 命令（分析历史会话：使用模式、高频话题、工具偏好、token 消耗趋势）

---

## N. 飞书渠道

- [ ] N01 实现飞书渠道基类（start/stop/receive/send）
- [ ] N02 接入飞书开放平台 Bot API
- [ ] N03 实现飞书消息接收（Webhook/长连接）
- [ ] N04 实现飞书文本消息发送
- [ ] N05 实现飞书富文本/Markdown 消息发送
- [ ] N06 实现飞书图片消息发送
- [ ] N07 实现飞书文件消息发送
- [ ] N08 实现飞书交互式卡片（确认操作按钮）
- [ ] N09 实现飞书群聊 @机器人 触发

---

## O. 微信渠道

- [ ] O01 实现微信渠道基类（start/stop/receive/send）
- [ ] O02 接入企业微信 API（或 wechaty 等框架）
- [ ] O03 实现微信文本消息接收
- [ ] O04 实现微信文本消息发送
- [ ] O05 实现微信图片消息发送
- [ ] O06 实现微信文件消息发送
- [ ] O07 实现消息长度限制（超 2000 字符自动分段）

---

## P. 记忆系统

### 上下文管理（工作记忆）

- [ ] P01 实现上下文窗口管理（系统提示 + 索引 + 对话 + 工具结果的组装）
- [ ] P02 实现对话历史滚动保留（最近 N 轮完整保留）
- [ ] P03 实现工具结果摘要化（旧的工具结果替换为一句话描述）
- [ ] P04 实现对话历史摘要化（早期对话用 LLM 生成摘要）
- [ ] P05 实现大附件替换（截图在历史中替换为文本描述）
- [ ] P06 实现上下文压缩策略（按优先级：工具摘要 → 对话摘要 → 记忆卸载 → 截断）
- [ ] P07 实现会话历史 JSONL 归档（会话结束后逐条写入 .jsonl 文件）
- [ ] P08 实现 JSONL 会话记录读取（按会话列表、时间范围检索历史会话）
- [ ] P09 实现 JSONL 会话回放（加载历史会话并恢复上下文）

### 长期记忆（持久存储）

- [ ] P10 设计记忆文件存储方案（Markdown + frontmatter）
- [ ] P11 实现记忆文件读写（创建/读取/更新/删除 .md 文件）
- [ ] P12 实现 MEMORY.md 索引管理（自动生成和更新索引）
- [ ] P13 实现 user 类型记忆写入
- [ ] P14 实现 feedback 类型记忆写入（含 Why + How to apply）
- [ ] P15 实现 project 类型记忆写入（含时间标注）
- [ ] P16 实现 reference 类型记忆写入
- [ ] P17 实现全局记忆（~/.proteus/memory/）和项目记忆的分层管理
- [ ] P18 实现记忆相关性判断（根据用户消息读取相关记忆文件）

### Autodream（自动沉淀）

- [ ] P19 实现 autodream 触发机制（每 N 轮对话触发）
- [ ] P20 实现 autodream 会话结束触发
- [ ] P21 实现 autodream 上下文压缩前触发
- [ ] P22 实现 autodream 提示词（回顾对话 → 判断是否有值得记忆的内容）
- [ ] P23 实现 autodream 异步执行（独立 LLM 调用，不阻塞主流程）
- [ ] P24 实现 autodream 输出解析（create/update/validate/delete 决策）
- [ ] P25 实现 autodream 结果应用（写入/更新/删除记忆文件 + 更新索引）
- [ ] P26 实现 autodream 失败容错（失败不影响主流程，仅记录日志）

### 记忆工具

- [ ] P27 实现 memory_save 工具（用户显式保存记忆）
- [ ] P28 实现 memory_recall 工具（检索记忆）
- [ ] P29 实现 memory_list 工具（列出所有记忆）
- [ ] P30 实现 memory_delete 工具（删除记忆）
- [ ] P31 实现实时检测写入（用户纠正/肯定时自动触发）

---

## Q. 测试

- [ ] Q01 编写数据模型单元测试（C01-C07）
- [ ] Q02 编写 LLM 层单元测试（mock Provider）
- [ ] Q03 编写工具系统单元测试（注册、参数解析、执行）
- [ ] Q04 编写文件工具单元测试
- [ ] Q05 编写 Shell 工具单元测试
- [ ] Q06 编写 Agent Loop 单元测试（固定 LLM 响应的完整流程）
- [ ] Q07 编写 CLI 渠道单元测试
- [ ] Q08 编写飞书渠道单元测试（mock API）
- [ ] Q09 编写微信渠道单元测试（mock API）
- [ ] Q10 编写浏览器工具集成测试（标记 @pytest.mark.integration）
- [ ] Q11 编写电脑操控集成测试（标记 @pytest.mark.integration）
- [ ] Q12 编写 Hook 机制单元测试（事件触发、拦截、超时）
- [ ] Q13 编写记忆系统单元测试（记忆文件读写、索引管理）
- [ ] Q14 编写上下文管理单元测试（压缩策略、摘要化、附件替换）
- [ ] Q15 编写 autodream 单元测试（触发、异步执行、结果应用）
- [ ] Q16 编写记忆相关性判断单元测试

---

## R. 扩展（长期）

- [ ] R01 插件系统（用户自定义工具注册）
- [ ] R02 多 Agent 协作
- [ ] R03 语音输入/输出
- [ ] R04 移动端操控（Android/iOS 模拟器）
- [ ] R05 本地模型支持（Ollama）
