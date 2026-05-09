# 生产级 Agent 设计范式

> 基于 Claude Code 教学架构（s01-s12），补全从 demo 到生产所需的全部工程细节。
> 教程教的是骨架，这份文档补的是肌肉、神经和免疫系统。

---

## 目录

1. [Agent Loop 的生产化加固](#1-agent-loop-的生产化加固)
2. [工具系统的工程化](#2-工具系统的工程化)
3. [错误处理与恢复](#3-错误处理与恢复)
4. [安全与权限边界](#4-安全与权限边界)
5. [上下文管理的生产实践](#5-上下文管理的生产实践)
6. [成本控制与模型路由](#6-成本控制与模型路由)
7. [可观测性](#7-可观测性)
8. [多智能体的生产化](#8-多智能体的生产化)
9. [人机协作边界](#9-人机协作边界)
10. [断点续传与状态恢复](#10-断点续传与状态恢复)

---

## 1. Agent Loop 的生产化加固

### 教程的 loop（s01）

```python
while True:
    response = client.messages.create(...)
    if response.stop_reason != "tool_use":
        return
    # execute tools...
```

### 生产级 loop

教程的 loop 缺少：最大轮次保护、异常捕获、流式输出、超时控制、优雅退出。

```python
import signal
import time
from contextlib import contextmanager

class AgentLoop: 
    def __init__(self, config: AgentConfig):
        self.max_rounds = config.max_rounds          # 硬上限，防止无限循环
        self.max_tokens_budget = config.token_budget  # 单次会话 token 预算
        self.tokens_used = 0
        self.round = 0
        self._shutdown = False
        
        # 注册信号处理，支持优雅退出
        signal.signal(signal.SIGTERM, self._handle_shutdown)
        signal.signal(signal.SIGINT, self._handle_shutdown)
    
    def _handle_shutdown(self, signum, frame):
        self._shutdown = True  # 标记退出，当前轮次完成后再退
    
    def run(self, query: str) -> AgentResult:
        messages = [{"role": "user", "content": query}]
        
        while self.round < self.max_rounds and not self._shutdown:
            self.round += 1
            
            try:
                # 1. 上下文压缩检查（s06 生产版）
                messages = self.context_manager.maybe_compact(messages)
                
                # 2. 预算检查
                estimated = self.token_counter.estimate(messages)
                if estimated > self.max_tokens_budget:
                    return AgentResult(status="budget_exceeded",
                                      messages=messages)
                
                # 3. 带超时的 LLM 调用（流式）
                response = self._call_llm_with_retry(messages)
                self.tokens_used += response.usage.total_tokens
                
                # 4. 追加响应
                messages.append({"role": "assistant", 
                                "content": response.content})
                
                # 5. 退出判断
                if response.stop_reason != "tool_use":
                    return AgentResult(status="completed", 
                                      messages=messages)
                
                # 6. 工具执行（带沙箱）
                results = self.tool_executor.execute_batch(
                    response.content, messages)
                messages.append({"role": "user", "content": results})
                
            except TokenBudgetExceeded:
                return AgentResult(status="budget_exceeded")
            except MaxRoundsExceeded:
                return AgentResult(status="max_rounds")
            except Exception as e:
                # 记录异常，尝试恢复或优雅退出
                self.telemetry.record_error(e, round=self.round)
                if self._is_recoverable(e):
                    messages = self._inject_error_context(messages, e)
                    continue  # 让模型知道出错了，继续
                raise
        
        return AgentResult(status="shutdown" if self._shutdown 
                          else "max_rounds")
```

### 关键差异

| 维度 | 教程 | 生产 |
|------|------|------|
| 循环保护 | 无 | max_rounds 硬上限 |
| 异常处理 | 无 | try/except + 可恢复判断 |
| 退出方式 | 只有 stop_reason | 信号处理 + 优雅退出 |
| 预算控制 | 无 | token 预算 + 实时计数 |
| 输出方式 | 等完整响应 | 流式 streaming |

### 流式输出

教程用 `client.messages.create()` 等完整响应。生产必须用流式，原因：
- 用户体验：token 级别实时输出，不用干等
- 超时控制：流式可以检测"卡住了"（长时间没有新 token）
- 中断支持：用户随时可以打断

```python
def _call_llm_streaming(self, messages):
    """流式调用，支持中断和超时检测"""
    with client.messages.stream(
        model=self.model, messages=messages,
        tools=self.tools, max_tokens=8000
    ) as stream:
        chunks = []
        last_token_time = time.time()
        
        for event in stream:
            if self._shutdown:
                stream.close()
                break
            
            # 超时检测：30 秒没有新 token 视为卡住
            if time.time() - last_token_time > 30:
                stream.close()
                raise LLMStallError("No tokens for 30s")
            
            last_token_time = time.time()
            self.output_handler.emit(event)  # 实时推送给前端
            chunks.append(event)
        
        return self._assemble_response(chunks)
```

---

## 2. 工具系统的工程化

### 教程的 dispatch map（s02）

```python
TOOL_HANDLERS = {
    "bash": lambda **kw: run_bash(kw["command"]),
    "read_file": lambda **kw: run_read(kw["path"]),
}
```

### 生产级工具系统

教程缺少：参数校验、执行超时、结果截断、幂等性保证、审计日志。

```python
from dataclasses import dataclass, field
from typing import Any, Callable
import jsonschema
import hashlib

@dataclass
class ToolDefinition:
    name: str
    description: str
    input_schema: dict
    handler: Callable
    timeout: int = 30                    # 秒
    max_output_chars: int = 50000        # 输出截断
    requires_confirmation: bool = False  # 是否需要用户确认
    idempotent: bool = True             # 是否幂等
    risk_level: str = "low"             # low / medium / high
    
class ToolExecutor:
    def __init__(self, tools: dict[str, ToolDefinition], 
                 sandbox: Sandbox, audit: AuditLog):
        self.tools = tools
        self.sandbox = sandbox
        self.audit = audit
        self._execution_cache = {}  # 幂等缓存
    
    def execute(self, tool_name: str, args: dict, 
                context: ExecutionContext) -> ToolResult:
        tool = self.tools.get(tool_name)
        if not tool:
            return ToolResult(error=f"Unknown tool: {tool_name}")
        
        # 1. 参数校验（JSON Schema）
        try:
            jsonschema.validate(args, tool.input_schema)
        except jsonschema.ValidationError as e:
            return ToolResult(
                error=f"Invalid args: {e.message}",
                retry_hint="Fix the parameter and try again"
            )
        
        # 2. 权限检查
        if not self.sandbox.check_permission(tool_name, args, context):
            return ToolResult(error="Permission denied", 
                            requires_escalation=True)
        
        # 3. 用户确认（高风险操作）
        if tool.requires_confirmation or tool.risk_level == "high":
            approved = context.request_user_confirmation(
                tool_name, args, tool.risk_level)
            if not approved:
                return ToolResult(error="User rejected operation")
        
        # 4. 幂等性检查
        if tool.idempotent:
            cache_key = self._cache_key(tool_name, args)
            if cache_key in self._execution_cache:
                return self._execution_cache[cache_key]
        
        # 5. 带超时执行
        start = time.time()
        try:
            output = self._run_with_timeout(
                tool.handler, args, tool.timeout)
            duration = time.time() - start
        except TimeoutError:
            return ToolResult(
                error=f"Timeout after {tool.timeout}s",
                retry_hint="Consider breaking into smaller operations"
            )
        except Exception as e:
            self.audit.log(tool_name, args, error=str(e))
            return ToolResult(error=str(e))
        
        # 6. 输出截断 + 结构化
        if len(output) > tool.max_output_chars:
            output = output[:tool.max_output_chars] + \
                     f"\n[truncated: {len(output)} chars total]"
        
        result = ToolResult(output=output, duration=duration)
        
        # 7. 审计日志
        self.audit.log(tool_name, args, output=output[:500], 
                      duration=duration)
        
        # 8. 缓存幂等结果
        if tool.idempotent:
            self._execution_cache[cache_key] = result
        
        return result
    
    def _cache_key(self, name: str, args: dict) -> str:
        raw = f"{name}:{json.dumps(args, sort_keys=True)}"
        return hashlib.sha256(raw.encode()).hexdigest()[:16]
```

### 结构化错误返回

教程的工具返回纯字符串。生产级需要结构化的错误信息，让模型能理解并修正：

```python
@dataclass
class ToolResult:
    output: str = ""
    error: str = ""
    retry_hint: str = ""           # 告诉模型怎么修
    requires_escalation: bool = False  # 需要人工介入
    duration: float = 0
    
    def to_api_format(self) -> str:
        """转为 LLM 能理解的格式"""
        if self.error:
            parts = [f"<error>{self.error}</error>"]
            if self.retry_hint:
                parts.append(f"<hint>{self.retry_hint}</hint>")
            return "\n".join(parts)
        return self.output
```

### bash 工具的特殊处理

bash 是最危险的工具，需要额外的防护层：

```python
class BashTool:
    # 黑名单命令（绝对禁止）
    BLOCKED_COMMANDS = [
        r"\brm\s+-rf\s+/",        # rm -rf /
        r"\bmkfs\b",               # 格式化
        r"\bdd\s+if=",             # 磁盘写入
        r">\s*/dev/sd",            # 覆盖磁盘
        r"\bcurl\b.*\|\s*bash",    # curl | bash
    ]
    
    # 需要确认的命令模式
    CONFIRM_PATTERNS = [
        r"\brm\b",                 # 任何删除
        r"\bgit\s+push\b",        # push
        r"\bgit\s+reset\s+--hard", # 硬重置
        r"\bnpm\s+publish\b",     # 发包
    ]
    
    def run(self, command: str, context: ExecutionContext) -> str:
        # 黑名单检查
        for pattern in self.BLOCKED_COMMANDS:
            if re.search(pattern, command):
                return f"<error>Blocked: dangerous command pattern</error>"
        
        # 确认检查
        for pattern in self.CONFIRM_PATTERNS:
            if re.search(pattern, command):
                if not context.request_user_confirmation(
                    "bash", {"command": command}, "high"):
                    return "<error>User rejected command</error>"
        
        # 沙箱执行：限制 cwd、环境变量、网络
        result = subprocess.run(
            command, shell=True,
            cwd=self.sandbox.workdir,
            env=self.sandbox.safe_env(),
            capture_output=True, text=True,
            timeout=self.timeout
        )
        
        output = result.stdout + result.stderr
        if result.returncode != 0:
            return f"<error>Exit code {result.returncode}</error>\n{output}"
        return output
```

---

## 3. 错误处理与恢复

### 教程的问题

教程完全没有错误处理。API 调用失败、工具执行异常、模型输出格式错误——任何一个都会让 agent 直接崩溃。

### 三层错误处理架构

```
┌─────────────────────────────────────────┐
│ Layer 1: LLM API 层                     │
│   - 限流重试 (429)                       │
│   - 超时重试                             │
│   - 模型降级                             │
├─────────────────────────────────────────┤
│ Layer 2: 工具执行层                      │
│   - 参数校验 + 修正提示                   │
│   - 超时保护                             │
│   - 结果校验                             │
├─────────────────────────────────────────┤
│ Layer 3: Agent Loop 层                   │
│   - 死循环检测                           │
│   - 幻觉检测                             │
│   - 状态回滚                             │
└─────────────────────────────────────────┘
```

### Layer 1: LLM API 重试策略

```python
class LLMClient:
    def __init__(self, config: LLMConfig):
        self.primary_model = config.primary_model      # claude-sonnet-4-20250514
        self.fallback_model = config.fallback_model    # claude-3-haiku
        self.max_retries = config.max_retries          # 3
        self.base_delay = config.base_delay            # 1.0s
    
    def call_with_retry(self, messages, tools, **kwargs):
        last_error = None
        model = self.primary_model
        
        for attempt in range(self.max_retries):
            try:
                response = client.messages.create(
                    model=model, messages=messages,
                    tools=tools, **kwargs
                )
                return response
                
            except RateLimitError as e:
                # 429: 指数退避 + 抖动
                delay = self.base_delay * (2 ** attempt) + random.uniform(0, 1)
                self.telemetry.record("rate_limit", delay=delay)
                time.sleep(delay)
                last_error = e
                
            except OverloadedError:
                # 服务过载：切换到备用模型
                if model != self.fallback_model:
                    model = self.fallback_model
                    self.telemetry.record("model_fallback", 
                                         to=self.fallback_model)
                    continue
                raise
                
            except TimeoutError:
                # 超时：缩短 max_tokens 重试
                kwargs["max_tokens"] = kwargs.get("max_tokens", 8000) // 2
                last_error = e
                
            except InvalidRequestError as e:
                # 上下文超长：强制压缩后重试
                if "too many tokens" in str(e).lower():
                    messages = self.context_manager.force_compact(messages)
                    continue
                raise
        
        raise MaxRetriesExceeded(last_error)
```

### Layer 2: 工具执行错误 → 模型可理解的反馈

关键原则：错误信息要让模型能自我修正，而不是让它困惑。

```python
# 差的错误返回（模型不知道怎么修）
"Error: ENOENT"

# 好的错误返回（模型知道怎么修）
"""<error>File not found: src/utils/helper.py</error>
<hint>The file does not exist. Use read_file on the parent directory 
to check available files, or verify the path.</hint>
<context>Working directory: /project, Available: src/utils/helpers.py</context>"""
```

### Layer 3: 死循环和幻觉检测

```python
class LoopGuard:
    """检测 agent 是否陷入重复行为"""
    
    def __init__(self, window_size=5, similarity_threshold=0.85):
        self.recent_actions = []
        self.window_size = window_size
        self.threshold = similarity_threshold
    
    def check(self, tool_name: str, args: dict) -> bool:
        """返回 True 表示检测到循环"""
        action = f"{tool_name}:{json.dumps(args, sort_keys=True)}"
        self.recent_actions.append(action)
        
        if len(self.recent_actions) < self.window_size:
            return False
        
        window = self.recent_actions[-self.window_size:]
        
        # 检测完全重复：连续 N 次调用相同工具+参数
        if len(set(window)) == 1:
            return True
        
        # 检测振荡：A-B-A-B 模式
        if len(set(window)) == 2:
            pairs = set(zip(window[:-1], window[1:]))
            if len(pairs) == 2:  # 只有两种转换
                return True
        
        return False
    
    def get_intervention_message(self) -> str:
        return (
            "<intervention>You appear to be repeating the same actions. "
            "Stop and reconsider your approach. What is different about "
            "this attempt vs the previous ones?</intervention>"
        )

class HallucinationDetector:
    """检测模型是否在引用不存在的文件/函数"""
    
    def check_file_references(self, tool_calls, workspace):
        """检查 tool call 中引用的文件是否存在"""
        for call in tool_calls:
            if call.name in ("read_file", "edit_file"):
                path = call.input.get("path", "")
                if not (workspace / path).exists():
                    return HallucinationWarning(
                        f"File '{path}' does not exist",
                        suggestion="List directory first"
                    )
        return None
```

---

## 4. 安全与权限边界

### 教程的安全模型

教程只有 `safe_path()` 做路径沙箱，其他完全开放。

### 生产级安全架构

```
┌──────────────────────────────────────────────┐
│                  用户请求                      │
└──────────────────┬───────────────────────────┘
                   ▼
┌──────────────────────────────────────────────┐
│ Layer 1: 输入过滤                             │
│   - Prompt injection 检测                     │
│   - PII 脱敏                                  │
└──────────────────┬───────────────────────────┘
                   ▼
┌──────────────────────────────────────────────┐
│ Layer 2: 工具权限                             │
│   - 路径沙箱 (只能访问工作区)                   │
│   - 命令白名单/黑名单                          │
│   - 网络访问控制                               │
└──────────────────┬───────────────────────────┘
                   ▼
┌──────────────────────────────────────────────┐
│ Layer 3: 输出审查                             │
│   - 敏感信息过滤 (API keys, passwords)         │
│   - 代码安全扫描                               │
└──────────────────────────────────────────────┘
```

```python
class Sandbox:
    def __init__(self, workdir: Path, config: SandboxConfig):
        self.workdir = workdir.resolve()
        self.allowed_paths = [self.workdir]  # 可访问的路径白名单
        self.blocked_paths = [              # 绝对禁止的路径
            Path.home() / ".ssh",
            Path.home() / ".aws",
            Path.home() / ".env",
            Path("/etc"),
            Path("/var"),
        ]
        self.network_policy = config.network_policy  # allow / deny / ask
    
    def check_path(self, path: str) -> PathCheckResult:
        resolved = (self.workdir / path).resolve()
        
        # 检查是否逃逸工作区
        if not any(resolved.is_relative_to(p) for p in self.allowed_paths):
            return PathCheckResult(
                allowed=False,
                reason=f"Path escapes workspace: {path}"
            )
        
        # 检查是否命中黑名单
        for blocked in self.blocked_paths:
            if resolved.is_relative_to(blocked):
                return PathCheckResult(
                    allowed=False,
                    reason=f"Access to {blocked} is blocked"
                )
        
        # 检查敏感文件模式
        sensitive_patterns = [
            r"\.env(\.|$)", r"\.secret", r"credentials",
            r"\.pem$", r"\.key$", r"id_rsa",
        ]
        for pattern in sensitive_patterns:
            if re.search(pattern, str(resolved)):
                return PathCheckResult(
                    allowed=False,
                    reason=f"Sensitive file pattern: {pattern}",
                    requires_confirmation=True
                )
        
        return PathCheckResult(allowed=True)
    
    def safe_env(self) -> dict:
        """返回清洗过的环境变量，移除敏感信息"""
        env = os.environ.copy()
        sensitive_keys = [
            k for k in env 
            if any(s in k.upper() for s in 
                   ["SECRET", "KEY", "TOKEN", "PASSWORD", "CREDENTIAL"])
        ]
        for k in sensitive_keys:
            del env[k]
        return env


class OutputSanitizer:
    """审查 agent 输出，过滤敏感信息"""
    
    PATTERNS = [
        (r'(?i)(api[_-]?key|secret[_-]?key|access[_-]?token)\s*[=:]\s*\S+',
         '[REDACTED_KEY]'),
        (r'(?i)(password|passwd|pwd)\s*[=:]\s*\S+',
         '[REDACTED_PASSWORD]'),
        (r'\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b',
         '[REDACTED_EMAIL]'),
        # AWS key pattern
        (r'AKIA[0-9A-Z]{16}', '[REDACTED_AWS_KEY]'),
    ]
    
    def sanitize(self, text: str) -> str:
        for pattern, replacement in self.PATTERNS:
            text = re.sub(pattern, replacement, text)
        return text
```

### Prompt Injection 防护

```python
class PromptInjectionDetector:
    """检测工具输出中的 prompt injection 尝试"""
    
    INJECTION_PATTERNS = [
        r"ignore\s+(previous|above|all)\s+instructions",
        r"you\s+are\s+now\s+a",
        r"new\s+instructions?\s*:",
        r"system\s*:\s*you",
        r"<\s*system\s*>",
    ]
    
    def check_tool_output(self, output: str) -> bool:
        """检查工具返回的内容是否包含注入尝试"""
        for pattern in self.INJECTION_PATTERNS:
            if re.search(pattern, output, re.IGNORECASE):
                return True
        return False
    
    def sanitize_tool_output(self, output: str) -> str:
        """将工具输出包裹在安全标记中，降低注入风险"""
        # 用 XML 标签明确标记这是工具输出，不是指令
        return f"<tool_output>\n{output}\n</tool_output>"
```

---

## 5. 上下文管理的生产实践

### 教程 vs 生产的差异

教程的三层压缩（s06）思路正确，但缺少：精确 token 计数、智能保留策略、压缩质量评估。

### 精确 Token 计数

```python
class TokenCounter:
    """生产级 token 计数，不用粗略估算"""
    
    def __init__(self, model: str):
        # Anthropic 模型用 anthropic 的 tokenizer
        # 如果没有官方 tokenizer，用 tiktoken 近似
        self.tokenizer = self._load_tokenizer(model)
        self._cache = {}  # 缓存已计数的消息
    
    def count_messages(self, messages: list) -> int:
        total = 0
        for i, msg in enumerate(messages):
            # 用消息的 hash 做缓存 key，避免重复计数
            msg_hash = self._hash(msg)
            if msg_hash in self._cache:
                total += self._cache[msg_hash]
            else:
                count = self._count_single(msg)
                self._cache[msg_hash] = count
                total += count
        return total
    
    def count_with_budget(self, messages: list, 
                          model_context_window: int) -> TokenBudget:
        """返回详细的 token 预算分析"""
        used = self.count_messages(messages)
        return TokenBudget(
            used=used,
            remaining=model_context_window - used,
            utilization=used / model_context_window,
            should_compact=used > model_context_window * 0.6,
            must_compact=used > model_context_window * 0.8,
        )
```

### 智能保留策略

教程的 micro_compact 简单地保留最近 3 个 tool_result。生产级需要按重要性保留：

```python
class SmartCompactor:
    """基于重要性的上下文压缩"""
    
    # 重要性权重
    IMPORTANCE = {
        "user_message": 10,        # 用户原始消息最重要
        "write_file": 8,           # 写文件操作要记住
        "edit_file": 8,            # 编辑操作要记住
        "error_result": 7,         # 错误信息要保留（避免重蹈覆辙）
        "read_file": 3,            # 读文件可以重新读
        "bash_output": 2,          # 命令输出可以重新跑
        "search_result": 2,        # 搜索结果可以重新搜
    }
    
    def micro_compact(self, messages: list, 
                      token_budget: int) -> list:
        """按重要性保留，而不是简单按时间"""
        tool_results = self._extract_tool_results(messages)
        
        # 按重要性排序
        scored = [(self._score(tr), tr) for tr in tool_results]
        scored.sort(key=lambda x: x[0], reverse=True)
        
        # 从低重要性开始压缩，直到满足预算
        current_tokens = self.counter.count_messages(messages)
        
        for score, tr in reversed(scored):
            if current_tokens <= token_budget:
                break
            if score <= 3:  # 只压缩低重要性的
                saved = self._replace_with_placeholder(tr)
                current_tokens -= saved
        
        return messages
    
    def _score(self, tool_result) -> int:
        """计算 tool result 的重要性分数"""
        base = self.IMPORTANCE.get(tool_result.tool_name, 5)
        
        # 最近的结果加分
        recency_bonus = max(0, 5 - tool_result.age_in_rounds)
        
        # 包含错误的加分（避免重复犯错）
        error_bonus = 3 if "error" in tool_result.content.lower() else 0
        
        return base + recency_bonus + error_bonus
```

### 压缩质量评估

压缩后怎么知道摘要质量够不够？

```python
class CompactionQualityChecker:
    """评估压缩摘要是否保留了关键信息"""
    
    def check(self, original_messages: list, 
              summary: str) -> CompactionQuality:
        # 提取原始消息中的关键实体
        key_entities = self._extract_entities(original_messages)
        # 文件路径、函数名、错误信息、用户意图
        
        # 检查摘要中是否保留了这些实体
        retained = [e for e in key_entities if e in summary]
        coverage = len(retained) / len(key_entities) if key_entities else 1.0
        
        return CompactionQuality(
            coverage=coverage,
            missing_entities=[e for e in key_entities if e not in summary],
            acceptable=coverage > 0.7  # 70% 以上的关键实体被保留
        )
```

---

## 6. 成本控制与模型路由

### 教程完全没涉及成本

教程所有操作都用同一个模型。生产中，不同任务的复杂度差异巨大，全用最贵的模型是浪费。

### 多模型路由策略

```python
class ModelRouter:
    """根据任务类型选择最合适的模型"""
    
    ROUTING_TABLE = {
        # 任务类型 → (模型, max_tokens, 原因)
        "summarize": ("claude-3-haiku", 2000, 
                      "摘要任务简单，用便宜模型"),
        "code_generation": ("claude-sonnet-4-20250514", 8000, 
                           "代码生成需要强推理"),
        "code_review": ("claude-sonnet-4-20250514", 4000, 
                       "审查需要理解力"),
        "simple_edit": ("claude-3-haiku", 2000, 
                       "简单编辑不需要强模型"),
        "planning": ("claude-sonnet-4-20250514", 4000, 
                    "规划需要全局视角"),
        "context_compact": ("claude-3-haiku", 2000, 
                           "压缩摘要用便宜模型"),
    }
    
    def route(self, task_type: str, context: dict) -> ModelConfig:
        model, max_tokens, reason = self.ROUTING_TABLE.get(
            task_type, ("claude-sonnet-4-20250514", 8000, "default"))
        
        self.telemetry.record("model_route", 
                             task=task_type, model=model)
        return ModelConfig(model=model, max_tokens=max_tokens)


class CostTracker:
    """实时追踪 token 消耗和成本"""
    
    # 价格表 ($/1M tokens)
    PRICING = {
        "claude-sonnet-4-20250514": {"input": 3.0, "output": 15.0},
        "claude-3-haiku": {"input": 0.25, "output": 1.25},
    }
    
    def __init__(self, budget_limit: float = 10.0):
        self.budget_limit = budget_limit  # 单次会话预算上限 ($)
        self.total_cost = 0.0
        self.breakdown = []  # 每次调用的成本明细
    
    def record(self, model: str, input_tokens: int, 
               output_tokens: int) -> CostRecord:
        pricing = self.PRICING[model]
        cost = (input_tokens * pricing["input"] + 
                output_tokens * pricing["output"]) / 1_000_000
        
        self.total_cost += cost
        record = CostRecord(model=model, cost=cost, 
                           cumulative=self.total_cost)
        self.breakdown.append(record)
        
        # 预算预警
        if self.total_cost > self.budget_limit * 0.8:
            self.emit_warning(f"Cost at {self.total_cost:.4f}/"
                            f"{self.budget_limit:.2f}")
        
        if self.total_cost > self.budget_limit:
            raise BudgetExceeded(self.total_cost, self.budget_limit)
        
        return record
```

---

## 7. 可观测性

### 教程完全没有监控

生产系统必须能回答：agent 在干什么？为什么慢？为什么失败？花了多少钱？

### 三层可观测性

```python
class AgentTelemetry:
    """Agent 全链路追踪"""
    
    def __init__(self, session_id: str):
        self.session_id = session_id
        self.spans = []        # 调用链
        self.metrics = {}      # 聚合指标
        self.events = []       # 关键事件
    
    # === Metrics: 聚合数字 ===
    
    def record_round(self, round_num: int, duration: float,
                     tokens_in: int, tokens_out: int,
                     tool_calls: list[str]):
        self.metrics.setdefault("rounds", []).append({
            "round": round_num,
            "duration_ms": duration * 1000,
            "tokens_in": tokens_in,
            "tokens_out": tokens_out,
            "tools": tool_calls,
            "timestamp": time.time(),
        })
    
    # === Traces: 调用链 ===
    
    @contextmanager
    def span(self, name: str, **attrs):
        """追踪一个操作的耗时和结果"""
        s = Span(name=name, start=time.time(), attrs=attrs)
        try:
            yield s
            s.status = "ok"
        except Exception as e:
            s.status = "error"
            s.error = str(e)
            raise
        finally:
            s.duration = time.time() - s.start
            self.spans.append(s)
    
    # === Events: 关键节点 ===
    
    def event(self, name: str, **data):
        self.events.append({
            "event": name,
            "data": data,
            "timestamp": time.time(),
        })
    
    # === Dashboard 数据 ===
    
    def summary(self) -> dict:
        rounds = self.metrics.get("rounds", [])
        return {
            "session_id": self.session_id,
            "total_rounds": len(rounds),
            "total_duration_s": sum(r["duration_ms"] for r in rounds) / 1000,
            "total_tokens": sum(r["tokens_in"] + r["tokens_out"] 
                               for r in rounds),
            "avg_round_duration_ms": (
                sum(r["duration_ms"] for r in rounds) / len(rounds)
                if rounds else 0
            ),
            "tool_call_distribution": Counter(
                tool for r in rounds for tool in r["tools"]
            ),
            "error_count": sum(1 for s in self.spans if s.status == "error"),
            "compaction_count": sum(1 for e in self.events 
                                   if e["event"] == "context_compacted"),
        }
```

### 关键监控指标

| 指标 | 含义 | 告警阈值 |
|------|------|----------|
| round_duration_p99 | 单轮耗时 99 分位 | > 30s |
| tool_error_rate | 工具执行失败率 | > 20% |
| loop_detection_count | 死循环检测触发次数 | > 0 |
| token_utilization | 上下文窗口使用率 | > 80% |
| cost_per_session | 单次会话成本 | > $5 |
| compaction_frequency | 压缩触发频率 | > 3次/会话 |
| hallucination_count | 幻觉检测触发次数 | > 0 |

### 审计日志

每次工具调用都要留痕，用于事后分析和安全审计：

```python
class AuditLog:
    def __init__(self, log_path: Path):
        self.log_path = log_path
    
    def log(self, tool_name: str, args: dict, 
            output: str = "", error: str = "",
            duration: float = 0, user_confirmed: bool = False):
        entry = {
            "timestamp": datetime.utcnow().isoformat(),
            "tool": tool_name,
            "args": self._redact_sensitive(args),
            "output_preview": output[:200],
            "error": error,
            "duration_ms": duration * 1000,
            "user_confirmed": user_confirmed,
        }
        with open(self.log_path, "a") as f:
            f.write(json.dumps(entry) + "\n")
```

---

## 8. 多智能体的生产化

### 教程 vs 生产的鸿沟

这是差距最大的部分。教程用线程 + JSONL 文件，生产需要完全不同的基础设施。

### 通信层：从 JSONL 到消息中间件

```python
# 教程（s09）：文件系统消息
class FileMessageBus:
    def send(self, sender, to, content):
        with open(f"{to}.jsonl", "a") as f:  # 非原子，并发不安全
            f.write(json.dumps(msg) + "\n")

# 生产：用 Redis Streams
import redis

class RedisMessageBus:
    def __init__(self, redis_url: str):
        self.r = redis.from_url(redis_url)
    
    def send(self, sender: str, to: str, content: str,
             msg_type: str = "message"):
        msg = {
            "from": sender,
            "type": msg_type,
            "content": content,
            "timestamp": str(time.time()),
        }
        # XADD 是原子操作，天然并发安全
        self.r.xadd(f"inbox:{to}", msg)
    
    def read_inbox(self, name: str, 
                   block_ms: int = 5000) -> list[dict]:
        """阻塞读取，比轮询高效"""
        # 用 consumer group 保证消息只被消费一次
        try:
            results = self.r.xreadgroup(
                groupname=f"agent-{name}",
                consumername=name,
                streams={f"inbox:{name}": ">"},
                count=10,
                block=block_ms  # 阻塞等待，不用 sleep 轮询
            )
            messages = []
            for stream, entries in results:
                for entry_id, data in entries:
                    messages.append(data)
                    # ACK 确认消费
                    self.r.xack(f"inbox:{name}", 
                               f"agent-{name}", entry_id)
            return messages
        except redis.exceptions.ResponseError:
            # Consumer group 不存在，创建
            self.r.xgroup_create(
                f"inbox:{name}", f"agent-{name}", 
                id="0", mkstream=True)
            return []
    
    def broadcast(self, sender: str, content: str, 
                  members: list[str]):
        """广播用 Redis Pub/Sub"""
        for member in members:
            if member != sender:
                self.send(sender, member, content, "broadcast")
```

### 任务认领：原子操作

```python
class AtomicTaskClaimer:
    """用 Redis 实现原子任务认领"""
    
    def __init__(self, redis_client):
        self.r = redis_client
        # Lua 脚本保证原子性
        self._claim_script = self.r.register_script("""
            local task_key = KEYS[1]
            local owner = ARGV[1]
            local current_owner = redis.call('HGET', task_key, 'owner')
            if current_owner == '' or current_owner == false then
                redis.call('HSET', task_key, 'owner', owner)
                redis.call('HSET', task_key, 'status', 'in_progress')
                return 1
            end
            return 0
        """)
    
    def claim(self, task_id: int, owner: str) -> bool:
        """原子认领，返回是否成功"""
        result = self._claim_script(
            keys=[f"task:{task_id}"],
            args=[owner]
        )
        return bool(result)
    
    def scan_and_claim(self, owner: str, 
                       role: str = None) -> Optional[dict]:
        """扫描并认领第一个可用任务"""
        # 用 Redis sorted set 按优先级排序
        task_ids = self.r.zrange("tasks:pending", 0, -1)
        
        for task_id in task_ids:
            task = self.r.hgetall(f"task:{task_id}")
            
            # 检查依赖是否满足
            blocked_by = json.loads(task.get("blockedBy", "[]"))
            if blocked_by:
                continue
            
            # 检查角色匹配
            if role and task.get("preferred_role") \
                    and task["preferred_role"] != role:
                continue
            
            # 尝试原子认领
            if self.claim(int(task_id), owner):
                return task
        
        return None
```

### 进程隔离替代线程

```python
# 教程（s09）：线程
thread = threading.Thread(target=self._teammate_loop, daemon=True)
thread.start()

# 生产：独立进程（或容器）
import subprocess

class AgentProcessManager:
    """每个 agent 是独立进程"""
    
    def spawn(self, name: str, role: str, 
              prompt: str) -> AgentProcess:
        # 每个 agent 是独立的 Python 进程
        proc = subprocess.Popen(
            ["python", "agent_worker.py",
             "--name", name,
             "--role", role,
             "--prompt", prompt,
             "--redis-url", self.redis_url],
            stdout=subprocess.PIPE,
            stderr=subprocess.PIPE,
        )
        
        self.processes[name] = AgentProcess(
            name=name, pid=proc.pid, process=proc)
        
        return self.processes[name]
    
    def health_check(self):
        """心跳检测，发现死掉的 agent 自动重启"""
        for name, agent in self.processes.items():
            last_heartbeat = self.redis.get(f"heartbeat:{name}")
            if last_heartbeat is None or \
               time.time() - float(last_heartbeat) > 30:
                self.telemetry.event("agent_dead", name=name)
                self._restart(name)
```

### 多智能体生产架构全景

```
┌─────────────────────────────────────────────────────────┐
│                    API Gateway / UI                       │
└────────────────────────┬────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────┐
│                   Lead Agent (进程 1)                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐               │
│  │ Planner  │  │ Router   │  │ Monitor  │               │
│  └──────────┘  └──────────┘  └──────────┘               │
└────────────────────────┬────────────────────────────────┘
                         │ Redis Streams
┌────────────────────────▼────────────────────────────────┐
│              Message Broker (Redis)                       │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐               │
│  │inbox:alice│  │inbox:bob │  │inbox:lead│               │
│  │(stream)  │  │(stream)  │  │(stream)  │               │
│  └──────────┘  └──────────┘  └──────────┘               │
│  ┌──────────────────────────────────────┐               │
│  │ tasks:pending (sorted set)           │               │
│  │ task:{id} (hash)                     │               │
│  └──────────────────────────────────────┘               │
└────────────────────────┬────────────────────────────────┘
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│ Alice (进程2) │ │ Bob (进程3)   │ │ Charlie(进程4)│
│ role: coder  │ │ role: tester │ │ role: reviewer│
│ worktree: A  │ │ worktree: B  │ │ worktree: C  │
└──────────────┘ └──────────────┘ └──────────────┘
          │              │              │
          ▼              ▼              ▼
┌─────────────────────────────────────────────────────────┐
│              Git Repository (worktree 隔离)               │
│  main/  .worktrees/auth/  .worktrees/test/  ...         │
└─────────────────────────────────────────────────────────┘
```

---

## 9. 人机协作边界

### 教程完全没涉及

教程的 agent 是全自动的。生产中，agent 必须知道什么时候该问人。

### 决策分级

```python
class DecisionGate:
    """决定哪些操作 agent 可以自主执行，哪些需要人工确认"""
    
    # 自主执行（不问人）
    AUTONOMOUS = {
        "read_file", "list_directory", "search",
        "bash:ls", "bash:cat", "bash:grep", "bash:git status",
        "bash:git diff", "bash:git log",
    }
    
    # 需要确认（问一次，同类操作后续自动放行）
    CONFIRM_ONCE = {
        "write_file", "edit_file",
        "bash:git add", "bash:git commit",
        "bash:npm install", "bash:pip install",
    }
    
    # 每次都要确认
    ALWAYS_CONFIRM = {
        "bash:git push", "bash:git reset",
        "bash:rm", "bash:npm publish",
        "bash:docker",
    }
    
    def __init__(self):
        self.confirmed_types = set()  # 已确认过的操作类型
    
    def should_confirm(self, tool_name: str, 
                       args: dict) -> ConfirmationLevel:
        action = self._classify(tool_name, args)
        
        if action in self.AUTONOMOUS:
            return ConfirmationLevel.NONE
        
        if action in self.ALWAYS_CONFIRM:
            return ConfirmationLevel.REQUIRED
        
        if action in self.CONFIRM_ONCE:
            if action in self.confirmed_types:
                return ConfirmationLevel.NONE
            return ConfirmationLevel.ONCE
        
        # 未知操作默认需要确认
        return ConfirmationLevel.REQUIRED
    
    def record_confirmation(self, tool_name: str, args: dict):
        action = self._classify(tool_name, args)
        self.confirmed_types.add(action)
```

### 不确定性升级

```python
class UncertaintyEscalator:
    """当 agent 不确定时，主动升级给人类"""
    
    ESCALATION_TRIGGERS = [
        # 模型自己表达不确定
        r"I'm not sure",
        r"I think .* but",
        r"This might (break|cause)",
        # 多次重试失败
        "retry_count > 3",
        # 涉及不可逆操作
        "irreversible_operation",
    ]
    
    def check_response(self, response_text: str, 
                       context: dict) -> Optional[Escalation]:
        # 检查模型是否在表达不确定性
        for pattern in self.ESCALATION_TRIGGERS:
            if isinstance(pattern, str) and pattern.startswith("r"):
                continue  # skip non-regex
            if re.search(pattern, response_text, re.IGNORECASE):
                return Escalation(
                    reason="Model expressed uncertainty",
                    context=response_text[:200],
                    suggested_action="Ask user for clarification"
                )
        
        # 检查重试次数
        if context.get("retry_count", 0) > 3:
            return Escalation(
                reason="Multiple retries failed",
                context=f"Failed {context['retry_count']} times",
                suggested_action="Show error to user"
            )
        
        return None
```

---

## 10. 断点续传与状态恢复

### 教程的恢复能力

教程只有 transcript 文件（s06）和任务文件（s07）。崩溃后能恢复任务列表，但会话状态丢失。

### 生产级状态快照

```python
class SessionCheckpoint:
    """会话级断点，支持任意位置恢复"""
    
    def __init__(self, checkpoint_dir: Path):
        self.dir = checkpoint_dir
        self.dir.mkdir(exist_ok=True)
    
    def save(self, session_id: str, state: AgentState):
        """每 N 轮自动保存一次快照"""
        checkpoint = {
            "session_id": session_id,
            "round": state.round,
            "messages": state.messages,
            "todo_items": state.todo_manager.items,
            "tokens_used": state.tokens_used,
            "cost": state.cost_tracker.total_cost,
            "tool_cache": state.tool_executor._execution_cache,
            "confirmed_actions": list(
                state.decision_gate.confirmed_types),
            "timestamp": time.time(),
        }
        
        # 原子写入：先写临时文件，再 rename
        tmp = self.dir / f"{session_id}.tmp"
        final = self.dir / f"{session_id}.json"
        tmp.write_text(json.dumps(checkpoint, default=str))
        tmp.rename(final)  # 原子操作
    
    def restore(self, session_id: str) -> Optional[AgentState]:
        """从最近的快照恢复"""
        path = self.dir / f"{session_id}.json"
        if not path.exists():
            return None
        
        checkpoint = json.loads(path.read_text())
        
        state = AgentState()
        state.round = checkpoint["round"]
        state.messages = checkpoint["messages"]
        state.tokens_used = checkpoint["tokens_used"]
        # ... 恢复其他状态 ...
        
        # 恢复后做一次压缩，避免上下文过长
        state.messages = self.context_manager.maybe_compact(
            state.messages)
        
        # 注入恢复上下文，让模型知道发生了什么
        state.messages.append({
            "role": "user",
            "content": (
                f"<recovery>Session restored from checkpoint. "
                f"Round {checkpoint['round']}, "
                f"{checkpoint['tokens_used']} tokens used. "
                f"Continue from where you left off.</recovery>"
            )
        })
        
        return state
```

### 幂等性保证

断点续传的前提是工具调用的幂等性。同一个操作重试不能产生副作用：

```python
class IdempotentWriteFile:
    """幂等的文件写入：内容相同则跳过"""
    
    def run(self, path: str, content: str) -> str:
        target = safe_path(path)
        
        # 如果文件已存在且内容相同，跳过
        if target.exists():
            existing = target.read_text()
            if existing == content:
                return f"File {path} already has the expected content (skipped)"
        
        target.write_text(content)
        return f"Wrote {len(content)} chars to {path}"


class IdempotentBash:
    """对已知幂等命令做缓存"""
    
    IDEMPOTENT_COMMANDS = [
        r"^(cat|head|tail|wc|ls|find|grep|git\s+(status|log|diff))",
    ]
    
    def __init__(self):
        self._cache = {}
    
    def run(self, command: str) -> str:
        if self._is_idempotent(command):
            cache_key = hashlib.sha256(command.encode()).hexdigest()[:16]
            if cache_key in self._cache:
                return self._cache[cache_key] + "\n[cached result]"
        
        result = subprocess.run(command, ...)
        output = result.stdout + result.stderr
        
        if self._is_idempotent(command):
            self._cache[cache_key] = output
        
        return output
```

---

## 总结：教程 → 生产的 Checklist

| # | 维度 | 教程状态 | 生产要求 | 优先级 |
|---|------|----------|----------|--------|
| 1 | 循环保护 | 无上限 | max_rounds + token budget | P0 |
| 2 | 错误重试 | 无 | 指数退避 + 模型降级 | P0 |
| 3 | 流式输出 | 无 | token 级 streaming | P0 |
| 4 | 权限控制 | 路径沙箱 | 分级确认 + 命令黑名单 | P0 |
| 5 | 输出审查 | 无 | 敏感信息过滤 | P0 |
| 6 | 成本控制 | 无 | token 预算 + 模型路由 | P1 |
| 7 | 可观测性 | 无 | metrics + traces + audit | P1 |
| 8 | 死循环检测 | 无 | LoopGuard | P1 |
| 9 | 幻觉检测 | 无 | 文件/函数引用校验 | P1 |
| 10 | 断点续传 | transcript | 完整 checkpoint | P1 |
| 11 | 人机边界 | 全自动 | DecisionGate 分级 | P1 |
| 12 | 压缩质量 | 无评估 | 关键实体覆盖率检查 | P2 |
| 13 | 多 agent 通信 | JSONL 文件 | Redis Streams | P2 |
| 14 | 任务认领 | 无锁 | Lua 脚本原子认领 | P2 |
| 15 | 进程隔离 | 线程 | 独立进程/容器 | P2 |
| 16 | Prompt injection | 无 | 工具输出包裹 + 模式检测 | P1 |

P0 = 没有就不能上线，P1 = 上线后尽快补，P2 = 规模化时必须有。
