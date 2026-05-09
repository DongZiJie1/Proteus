# s10: Team Protocols (团队协议)

`s01 > s02 > s03 > s04 > s05 > s06 | s07 > s08 > s09 > [ s10 ] s11 > s12`

> *"队友之间要有统一的沟通规矩"* -- 一个 request-response 模式驱动所有协商。

## 问题

s09 中队友能干活能通信, 但缺少结构化协调:

**关机**: 直接杀线程会留下写了一半的文件和过期的 config.json。需要握手 -- 领导请求, 队友批准 (收尾退出) 或拒绝 (继续干)。

**计划审批**: 领导说 "重构认证模块", 队友立刻开干。高风险变更应该先过审。

两者结构一样: 一方发带唯一 ID 的请求, 另一方引用同一 ID 响应。

## 解决方案

```
Shutdown Protocol            Plan Approval Protocol
==================           ======================

Lead             Teammate    Teammate           Lead
  |                 |           |                 |
  |--shutdown_req-->|           |--plan_req------>|
  | {req_id:"abc"}  |           | {req_id:"xyz"}  |
  |                 |           |                 |
  |<--shutdown_resp-|           |<--plan_resp-----|
  | {req_id:"abc",  |           | {req_id:"xyz",  |
  |  approve:true}  |           |  approve:true}  |

Shared FSM:
  [pending] --approve--> [approved]
  [pending] --reject---> [rejected]

Trackers:
  shutdown_requests = {req_id: {target, status}}
  plan_requests     = {req_id: {from, plan, status}}
```

## 工作原理

1. 领导生成 request_id, 通过收件箱发起关机请求。

```python
shutdown_requests = {}

def handle_shutdown_request(teammate: str) -> str:
    req_id = str(uuid.uuid4())[:8]
    shutdown_requests[req_id] = {"target": teammate, "status": "pending"}
    BUS.send("lead", teammate, "Please shut down gracefully.",
             "shutdown_request", {"request_id": req_id})
    return f"Shutdown request {req_id} sent (status: pending)"
```

2. 队友收到请求后, 用 approve/reject 响应。

```python
if tool_name == "shutdown_response":
    req_id = args["request_id"]
    approve = args["approve"]
    shutdown_requests[req_id]["status"] = "approved" if approve else "rejected"
    BUS.send(sender, "lead", args.get("reason", ""),
             "shutdown_response",
             {"request_id": req_id, "approve": approve})
```

3. 计划审批遵循完全相同的模式。队友提交计划 (生成 request_id), 领导审查 (引用同一个 request_id)。

```python
plan_requests = {}

# 队友端: 提交计划
def handle_plan_submission(teammate: str, plan: str) -> str:
    req_id = str(uuid.uuid4())[:8]
    plan_requests[req_id] = {"from": teammate, "plan": plan, "status": "pending"}
    BUS.send(teammate, "lead", plan,
             "plan_approval_request", {"request_id": req_id})
    return f"Plan {req_id} submitted for review (status: pending)"

# 领导端: 审批计划
def handle_plan_review(request_id, approve, feedback=""):
    req = plan_requests[request_id]
    req["status"] = "approved" if approve else "rejected"
    BUS.send("lead", req["from"], feedback,
             "plan_approval_response",
             {"request_id": request_id, "approve": approve})
```

一个 FSM, 两种用途。同样的 `pending -> approved | rejected` 状态机可以套用到任何请求-响应协议上。

## 设计取舍与局限性

**无超时机制**: 请求发出后, 如果队友崩溃或卡死, `pending` 状态会永远挂着。领导不知道该重发还是该等。生产环境需要加超时:

```python
shutdown_requests[req_id] = {
    "target": teammate, "status": "pending",
    "created_at": time.time()
}

# 轮询时检查
for req_id, req in shutdown_requests.items():
    if req["status"] == "pending" and time.time() - req["created_at"] > 60:
        req["status"] = "timeout"
        # 可以选择重发或强制终止
```

**reject 之后没有后续策略**: 队友拒绝关机, 领导收到 `rejected` 就结束了。没有重试逻辑, 没有升级机制。实际场景中可能需要: 等一段时间再请求、强制关机作为兜底、或者通知用户介入。

**计划审批是非阻塞的**: 队友提交计划后不会停下来等审批结果, 它的 agent loop 继续跑。如果领导还没审批, 队友可能已经开始执行了。要做到真正的门控, 需要队友在提交计划后主动进入等待状态:

```python
# 队友提交计划后应该这样
def submit_plan_and_wait(plan_text):
    req_id = submit_plan(plan_text)  # 发送计划
    # 进入轮询, 等审批结果
    while plan_requests[req_id]["status"] == "pending":
        time.sleep(2)
        inbox = BUS.read_inbox(name)
        # 检查收件箱里有没有审批响应
        for msg in inbox:
            if msg.get("request_id") == req_id:
                return msg["approve"]
    return plan_requests[req_id]["status"] == "approved"
```

**request_id 碰撞风险**: `uuid4()[:8]` 只取了 8 个十六进制字符 (32 bit), 在小规模教学场景下碰撞概率极低。但如果系统长期运行、请求量大, 碰撞会导致状态覆盖。生产环境应该用完整 UUID 或加上时间戳前缀。

**无持久化**: `shutdown_requests` 和 `plan_requests` 都是内存字典。进程重启后全部丢失, 正在进行的协议握手会中断。s09 的 config.json 和 JSONL 收件箱至少落了盘, 但 s10 的协议状态没有。

## 补充理解

### 核心洞察: 为什么需要 request_id?

没有 request_id 的世界:

```
Lead: "alice, 关机吧"
Lead: "bob, 关机吧"
Alice: "好的, 同意关机"
```

这个 "好的" 是回复哪个请求的? 在异步消息系统里, 消息到达顺序不确定。如果领导同时给 3 个队友发了关机请求, 收到的响应必须能对应到具体是谁的、哪个请求的。

request_id 就是这个关联键。它把一来一回绑定成一个完整的事务:

```
请求: {req_id: "abc", action: "shutdown", target: "alice"}
响应: {req_id: "abc", approve: true}
         ^^^
         同一个 ID, 明确知道这是 alice 对关机请求的回复
```

这和 HTTP 请求-响应的关联、数据库事务的 transaction ID、甚至快递单号的作用完全一样。

### 两个协议的方向是反的

这个细节容易忽略:

```
关机协议:   Lead → Teammate  (领导发起, 队友响应)
计划审批:   Teammate → Lead  (队友发起, 领导响应)
```

但 FSM 完全一样。这说明 request-response 模式跟"谁是老板"无关, 它是一个对称的通信原语。任何一方都可以发起请求, 任何一方都可以响应。

类比: 公司里不只是老板给员工派活, 员工也会给老板提审批。表单格式一样, 只是发起方和审批方换了。

### 队友 approve 关机后到底怎么退出?

文档展示了状态更新和消息发送, 但没有展示队友的 agent loop 如何真正停下来。完整流程:

```
1. 队友收到 shutdown_request 消息 (通过收件箱)
2. 消息注入到队友的 messages[] 里
3. 队友的 LLM 看到消息, 决定调用 shutdown_response 工具 (approve=true)
4. 工具执行: 更新状态 + 发送响应消息给领导
5. 关键: 工具返回后, 队友的 agent loop 需要检测到 "我已经同意关机了"
6. 设置一个标志位, 让循环在下一轮 break 退出
```

```python
# 在队友的 agent loop 里
if tool_name == "shutdown_response" and args["approve"]:
    should_shutdown = True

# 循环末尾
if should_shutdown:
    self._set_status(name, "shutdown")
    break  # 退出 agent loop, 线程结束
```

这就是 s09 里 `daemon=True` 的问题在 s10 里被正式解决的地方: 不再粗暴杀线程, 而是队友自己决定何时退出。

### 类比: 餐厅打烊

把关机协议想象成餐厅打烊:

- **s09 的做法**: 老板直接拉闸断电。厨师正在炒的菜糊了, 服务员端着的盘子摔了。
- **s10 的做法**: 老板广播 "准备打烊"。厨师说 "等我把这盘菜炒完" (reject, 继续干), 或者 "好的, 我关火了" (approve, 收尾退出)。服务员说 "等我把这桌的单结了" (reject), 然后 "好了, 可以走了" (approve)。

计划审批就像厨师说 "老板, 今天想试个新菜, 要用贵的食材", 老板看了说 "行" 或 "别浪费钱"。

### 为什么 s10 没有改动 agent loop?

对比前面的章节:
- s03 改了 loop (加了 nag reminder 注入)
- s06 改了 loop (加了上下文压缩)
- s08 改了 loop (加了 drain_notifications)

s10 没有动 loop。协议完全通过新增工具 + 收件箱消息实现。这是一个重要的架构特性: **好的协议不需要改底层基础设施, 只需要在现有通信通道上定义新的消息格式和状态机。**

就像 HTTP 协议不需要改 TCP, WebSocket 握手不需要改 HTTP 服务器内核。s10 的协议跑在 s09 的消息总线上, 总线本身一行代码没动。

## 相对 s09 的变更

| 组件           | 之前 (s09)       | 之后 (s10)                           |
|----------------|------------------|--------------------------------------|
| Tools          | 9                | 12 (+shutdown_req/resp +plan)        |
| 关机           | 仅自然退出       | 请求-响应握手                        |
| 计划门控       | 无               | 提交/审查与审批                      |
| 关联           | 无               | 每个请求一个 request_id              |
| FSM            | 无               | pending -> approved/rejected         |

## 试一试

```sh
cd learn-claude-code
python agents/s10_team_protocols.py
```

试试这些 prompt (英文 prompt 对 LLM 效果更好, 也可以用中文):

1. `Spawn alice as a coder. Then request her shutdown.`
2. `List teammates to see alice's status after shutdown approval`
3. `Spawn bob with a risky refactoring task. Review and reject his plan.`
4. `Spawn charlie, have him submit a plan, then approve it.`
5. 输入 `/team` 监控状态
