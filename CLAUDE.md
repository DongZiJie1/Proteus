# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

Proteus 是一个通用 AI Agent 工具，定位类似 Claude Code / OpenClaw，但额外具备操控电脑桌面和浏览器的能力。入口为 CLI 和消息渠道（微信、飞书等）。

## 技术栈

- **语言**: Python 3.11+
- **包管理**: uv（不用 pip/poetry）
- **项目配置**: pyproject.toml（不使用 setup.py）
- **类型标注**: 全量 type hints，mypy strict 模式
- **代码风格**: ruff 格式化 + lint
- **测试**: pytest
- **异步**: 全链路 asyncio（agent loop、工具执行、渠道通信均为异步）

## 常用命令

```bash
# 安装依赖
uv sync

# 运行 CLI 入口
uv run proteus

# 代码检查
uv run ruff check .
uv run ruff format --check .

# 格式化
uv run ruff format .

# 类型检查
uv run mypy src/

# 运行全部测试
uv run pytest

# 运行单个测试
uv run pytest tests/test_xxx.py::test_func_name -v
```

## 架构设计

```
src/proteus/
├── __main__.py          # CLI 入口 (click/typer)
├── agent/               # Agent 核心循环
│   ├── loop.py          # 主 agent loop：接收输入 → 思考 → 调用工具 → 返回
│   ├── planner.py       # 任务规划与分解
│   └── memory.py        # 上下文管理（短期/长期记忆）
├── llm/                 # LLM 抽象层
│   ├── client.py        # 统一 LLM 接口（支持多 provider）
│   └── prompts.py       # 系统提示词管理
├── tools/               # 工具注册与执行
│   ├── registry.py      # 工具注册中心（装饰器注册）
│   ├── base.py          # 工具基类
│   ├── computer.py      # 电脑操控（屏幕截图、鼠标键盘、窗口管理）
│   ├── browser.py       # 浏览器操控（Playwright 驱动）
│   ├── file.py          # 文件读写、搜索
│   ├── shell.py         # 终端命令执行
│   └── web.py           # 网页抓取、搜索
├── channels/            # 消息渠道接入
│   ├── base.py          # 渠道基类（统一消息收发接口）
│   ├── cli.py           # CLI 交互渠道
│   ├── wechat.py        # 微信渠道
│   └── feishu.py        # 飞书渠道
├── config.py            # 配置管理（pydantic-settings）
└── utils/               # 通用工具函数
```

### 核心循环

```
用户输入 (Channel) → Agent Loop → LLM 推理 → 工具调用 → 结果返回 → 循环或结束
```

Agent loop 是核心，职责：
1. 维护对话上下文
2. 调用 LLM 获取下一步动作
3. 解析 LLM 输出，决定是调用工具还是直接回复
4. 执行工具并将结果注入上下文
5. 循环直到任务完成或达到限制

### 工具系统

- 每个工具继承 `ToolBase`，通过 `@register_tool` 装饰器注册
- 工具描述自动生成为 JSON Schema，供 LLM function calling 使用
- 工具执行统一走异步，带超时和错误处理
- computer 和 browser 工具是本项目核心差异点

### 渠道系统

- 每个渠道继承 `ChannelBase`，实现 `receive()` / `send()` 方法
- 渠道与 Agent Loop 通过消息队列解耦
- CLI 渠道支持流式输出
- 消息渠道（微信/飞书）支持富文本、图片、文件消息

## 开发约束

### 代码规范

- 所有函数和方法必须有 type hints，包括返回值
- 不用 `Any` 类型，除非有充分理由并加注释
- 字符串统一用 f-string
- 日志用 `loguru`，不用 `print` 和标准 `logging`
- 错误处理：自定义异常体系，不要裸 `except`
- 配置通过环境变量或 `.env` 文件，硬编码配置禁止提交

### Agent 设计原则

- LLM 调用必须支持重试和降级（provider 切换）
- 工具调用必须有超时机制，默认 30s
- Agent loop 必须有最大迭代次数限制，防止死循环
- 所有 LLM 交互记录完整 prompt 和 response，用于调试
- 工具执行失败时，将错误信息返回给 LLM 让其自主决策，不要自动重试

### 电脑/浏览器操控

- 电脑操控基于 `pyautogui` + 截图（Pillow），操作前先截图确认当前状态
- 浏览器操控基于 `playwright`，优先用 accessibility tree 定位元素
- 每一步操控动作必须有日志记录，包含操作类型、目标、截图
- 涉及敏感操作（删除文件、发送消息）时，根据配置决定是否需要用户确认

### 渠道开发

- 新增渠道只需实现 `ChannelBase` 的 `receive()` / `send()` / `start()` / `stop()`
- 消息格式统一为内部 `Message` 模型，渠道负责与平台协议互转
- 渠道层不做任何业务逻辑，仅负责消息收发

### 不要做的事

- 不要引入 LangChain、LlamaIndex 等重型框架
- 不要用 Selenium（用 Playwright）
- 不要在工具层直接调 LLM
- 不要在渠道层处理业务逻辑
- 不要使用同步 I/O，全部异步
- 不要硬编码 API key 或密钥

## 依赖管理

核心依赖（初定）：
- `anthropic` / `openai` — LLM 调用
- `playwright` — 浏览器控制
- `pyautogui` — 桌面控制
- `pydantic` / `pydantic-settings` — 数据模型与配置
- `click` / `typer` — CLI
- `loguru` — 日志
- `httpx` — HTTP 客户端
- `rich` — 终端美化输出

新增依赖前先评估：是否活跃维护、是否轻量、是否有替代方案。核心路径上避免引入不稳定的库。

## 测试策略

- 工具层：mock LLM 调用，测试工具注册、参数解析、执行逻辑
- Agent loop：用固定 LLM 响应测试完整流程
- 渠道层：mock 平台 API，测试消息收发
- 集成测试：可选，默认跳过需要真实 API key 的测试（`@pytest.mark.integration`）

## 配置管理

使用 pydantic-settings，配置项包括：
- `PROTEUS_LLM_PROVIDER` — LLM 提供商
- `PROTEUS_LLM_API_KEY` — API 密钥
- `PROTEUS_LLM_MODEL` — 模型名称
- `PROTEUS_LOG_LEVEL` — 日志级别
- `PROTEUS_CHANNEL` — 启动的渠道
- `PROTEUS_COMPUTER_CONTROL_ENABLED` — 是否启用电脑操控
- `PROTEUS_BROWSER_ENABLED` — 是否启用浏览器操控
