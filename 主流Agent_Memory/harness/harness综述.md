## 2. Harness Engineering

Harness工程一定是2026年需要关注的方向。人们慢慢开始研究如何让AI能够长时间完成更加复杂的任务，而不受到上下文腐化的影响。这就是一种发展的趋势，从人们关注单次模型输入输出的结果（Prompt 工程），到多轮对话（Context 工程）上下文的高效组装。到现在，人们希望能够睡一觉，让AI一晚上不断运行，第二天有一个稳定良好的结果，如何达成这一目标，就是Harness工程要解决的事。从这里也可以看出行业的发展，人们不满足于只让AI做一个小任务，而是想要AI能够持续工作数小时甚至数天，全自动完成一个复杂的任务。

另外，也有多个同学反应这个问题面试会考察到，从面试的角度，也是需要掌握的。

最后想说的是，其实现在（2026年4月），Harness工程它更多的是一个思路和思想，什么是最佳的实现，其实没有一个最统一的标准。比如Anthropic，Openai，他们有自己的harness项目和实践。我们在这一节的笔记中也会带大家看2个Harness工程的实战：Anthropic和Clawcode。重要的是理解思想，而不是将他们奉为信条。然后自己实践，自己做项目的时候用上这些思想，设计Harness约束，使用Vibecoding来完成项目。

### 2.1 概念定义
**一句话定义**

Harness Engineering 是围绕 AI 模型构建"运行环境与管控基础设施"的工程实践，目的是让 AI Agent 在长时、复杂、多步骤的真实任务中稳定、可靠、高效地运行。

**核心思想**

AI 模型本身只是"原材料"——就像一颗裸露的 CPU、一台没有车身的引擎、一个没有厨房的天才厨师。模型有多聪明不再是瓶颈，环境有多可靠才是关键。 Harness Engineering 要解决的核心问题是：

* Context Rot（上下文腐化）：模型在长任务中记忆窗口被垃圾信息淹没，丧失对原始指令的遵循
* Hallucinated Completion（幻觉完成）：模型不堪重负时伪造"任务完成"的假象
* Model Drift（模型漂移）：多步执行中逐渐偏离初始目标

**类比**

举个例子帮大家理解：以一个计算机的运行架构类比，

* AI 模型 = CPU（原始算力）
* 上下文窗口 = RAM（有限的易失工作内存）
* Agent Harness = 操作系统（调度资源、管理内存、提供标准运行环境）
* AI Agent = 应用程序（在操作系统上运行的具体业务逻辑）

### 2.2 演进脉络
* Prompt Engineering（2023-2024）：核心问题是"对 AI 说什么"，人的角色是导演
* Context Engineering（2025）：核心问题是"让 AI 知道什么"，人的角色是图书管理员
* Harness Engineering（2026）：核心问题是"让 AI 在什么环境里做事"，人的角色是架构师
    三者是叠加关系：Harness 内部的每个 session 仍然需要 Context Engineering，每次 prompt 仍然需要 Prompt Engineering。

## 2.3 Harness 工程实践要点

i. **外部持久记忆**——用文件系统（JSON、Markdown、git log）代替模型内部记忆，解决"失忆"问题。
ii. **确定性验证轨道**——Linter、类型检查、单元测试等刚性门禁，AI 的输出必须通过测试才能被接受。
iii. **原子任务 + 上下文刷新**——每个 Agent 只做一个小任务就销毁，新 Agent 从干净状态启动，彻底杜绝 Context Rot。
iv. **子 Agent 集群（Swarm）**——用大量便宜快速的小模型并行工作，而非一个大模型顺序处理。
v. **技能文件（Skills）**——按需加载的专用指南，Agent 只看到与当前任务相关的工具和说明（不超过30个工具）。
vi. **Guard Rails + Checkpoints**——在关键节点自动运行检查，防止 Agent 偏离轨道。
vii. **Handoffs**——session 之间通过进度文件、git commit 等传递状态，实现跨 session 连续性。
viii. **Human-in-the-loop**——在关键阶段设置人工审核断点，弥补 AI 自验证的不足。
ix. **架构约束**——通过 ArchUnit 等框架强制要求 Agent 生成的代码遵循特定模式）。
x. **垃圾回收 Agent**——定期扫描修复代码库和文档中的不一致性，对抗系统熵增。

## 关键工程原则

- 轻量化，为删除而建：不要过度编码人类知识，模块要能快速移除替换（Manus 6个月5次重构、Vercel 删 80%手工工具反而更好）
- 模型无关性：Harness 不绑定特定模型，新模型出来能快速替换
- 将 Harness 视为数据集：每次 Agent 失败、漂移、异常都是珍贵的训练素材，用于迭代优化

### 2.4 Anthropic Harness架构详解
Anthropic 的 Harness 是目前行业内最被广泛引用的长时任务管控架构范例。我们来对它进行拆解，来看一下 Anthropic公司是如何使用Harness架构来完成一个长时，复杂的Vibecoding项目的。（其实我们这个笔记中的RAG项目也是从一个文档为起点，完成整个代码的。但是当时我们的方法比较朴素，用一个agent就做到底。学了本章，我们可以试一下，使用Harness工程，来完成这个项目，看看效果会不会更好）

#### 2.4.1设计哲学
Anthropic 的 Harness 核心设计理念是：与其让一个天才持续工作直到崩溃，不如让无数个"新手"接力完成，每人只干一件事。传统做法是把一个强大的模型（如 Claude Opus 4.6）扔进去从头干到尾——但无论模型多强，单一上下文窗口最终都会被垃圾数据淹没，造成上下文腐化，导致失忆和幻觉。
Anthropic 的方案彻底绕开了这个问题：**每个 Agent 只活一次，干完就死。**

#### 2.4.1总体结构
**1. App Spec / PRD：项目起点**
Anthropic 的 Harness 通常从一份由人提供的需求说明开始。这份需求说明不是直接交给模型去连续编码，而是作为整个系统初始化的输入。它的作用是定义产品目标、功能范围和预期结果，为后续的任务拆解和状态组织提供基础。也就是说，项目最开始进入系统的不是"代码任务"，而是一份高层次的目标描述。

**2. Initializer：初始化规划 Agent**
Initializer 的职责不是直接产出业务代码，而是把需求文档转化为后续开发可以执行的工程结构。它会读取需求，拆解出特性清单，建立任务边界，定义每个功能点的验证标准，并完成项目骨架和代码仓库的初始化。这个阶段的意义在于，先把一个模糊的产品目标整理成一个可以持续推进、可以被后续 agent 反复读取和执行的蓝图。

**3. Coding Loop：编码循环 Agent**
系统进入正式执行阶段后，**并不是由同一个 agent 一直写下去，而是不断启动新的 coding agent，让它们逐轮接力。** 每一轮 agent 都会先读取当前的任务状态和项目进度，然后只选择一个明确的小任务去实现。完成之后，它不会长期保留在系统中，而是会在交接完成后退出。下一轮再由一个全新的 agent 接手。Anthropic 正是通过这种循环式接力，而不是持续式长对话，来避免上下文不断膨胀的问题。

**4. External Memory：外部记忆系统**
Anthropic Harness 的一个核心思想，是将项目状态从模型内部记忆中剥离出来，放入外部可持久化的介质中，比如特性清单、进度文件和 Git 历史。这样一来，系统的连续性不依赖某个 agent 的上下文窗口，而依赖外部状态记录。每个新的 agent 启动时，只需要重新读取这些状态，就能接手上一个 agent 已经完成的工作，从而让整个系统具备持续推进的能力。

**5. Deterministic Validation：确定性验证轨道**
Anthropic 的架构并不把"是否完成"这件事交给模型自己判断，而是通过一套外部的、确定性的验证机制来裁定结果是否成立。也就是说，agent 生成代码之后，系统会通过测试、检查和验证来决定这项工作能否被接受。只有满足这些外部标准，系统才会更新状态并进入下一轮。这种设计的意义在于，用工程规则替代模型主观判断，把"感觉完成了"变成"验证通过了"。

**6. Sandbox / Isolation：隔离执行环境**
每个 agent 的运行通常都处于一个受控环境中，这样既能限制它可访问的工具范围，也能避免错误或脏上下文污染整个系统。隔离环境让每一轮执行都尽量在明确边界内进行，也让失败后的回滚、重试和重新启动变得可控。Anthropic Harness 依赖这种隔离性，把 agent 的自由度压缩在一个足够安全、足够清晰的工作空间里。

总结来说：Anthropic 公司的 Harness 架构核心，可以概括为一种"初始化拆解 + 外部状态持久化 + 单任务循环执行 + 确定性验证门控"的长时任务运行体系。它的关键不在于让单个大模型持续工作，而在于先由 Initializer 将需求拆解为可执行的特性清单，再让一个个全新的 Coding Agent 逐轮读取状态文件与代码库，只完成一个原子任务，并通过测试、提交与交接文件更新系统状态后立即退出。这样做的本质，是把项目记忆从模型内部迁移到外部系统，把任务完成判断从模型主观输出转移到确定性验证机制，从而显著降低上下文腐化、幻觉完成和多步任务失控的风险。

## 2.4.3 局限性

相比于单个Agent直接完成整个项目，Anhrpic 公司也承认整个工程的局限性（但我觉Harness Engineering还处于早期发展阶段(2026年3月写)，Anthropic 公司给大家打了个样，后续大家会继续在此基础上改进和发展，仍然有非常大的借鉴作用）

**Initializer 是单点高杠杆环节**

如果初始拆分错了，后面全部都会跟着错。也就是说，蓝图质量决定了整个系统上限。

**Handoff 质量并不稳定**

进度文件有时候总结得好，有时候会漏掉关键问题。一旦漏掉"为什么失败、怎么修复"，后面的 agent 可能反复踩同一个坑。

**多步可靠性仍然会衰减**

即使单步成功率很高，步骤一多，整体可靠性还是会下降。所以它不是彻底解决了可靠性问题，而是显著缓解。

**仍然需要 Human in the Loop**

从实战角度看，Anthropic Harness 更像：高度自动化，但不是完全放手，还是在关键节点引入人工检查。

## 2.2.4 参考资料

https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents

## 2.5 ClawCode Harness工程详解

Claude Code泄漏发生后，韩国开发者 Sigrid Jin 发起了 **Claw Code** 项目——用 AI 编排工具指挥多个 coding agent 并行工作，在 **3** 天内将 Claude Code 的核心功能从 TypeScript 重写为 Rust（48,599 行），产生了 292 个 commit。整个过程中人类几乎不写代码，只负责方向决策和任务分解。

这个项目破了Github涨星最快的记录，除了乘上了CC的热度外，而在于它完整展示了一套 **使用Harness工程进行 Vibe Coding的技巧** —— 如何用 AI 工厂模式完成大规模代码迁移同样值得学习。

这一节带大家快速掌握一下它的方法，值得提的是我没有非常深入地继续学习，而是快速了解了他的思路。如果你想更深入的学习，我也提供了它的代码地址，可以借鉴这个代码仓库，自己做一遍，感受Harness工程（毕竟 Harness 工程真的是需要自己去实践设计才能更好掌握的），也可以做一个Coding Agent，是一个不错的学习机会。

## 2.5.1 整体方法论：五阶段流水线

Claw Code 的核心思路是：**先把"什么需要被实现"变成机器可查的清单，再边实现边用 mock harness 自动验证行为对等性。**

整个流程分五个阶段：

```
阶段一：归档 + 快照 → 把原始 TS 代码表面提取为结构化 JSON（建立"真相基线"）
阶段二：Python 镜像工作区 → 1:1 结构占位 + 架构模型（理解原始系统）
阶段三：Parity Tracking → 量化覆盖率，进度可测量、可审计
阶段四：Mock Parity Harness → 确定性端到端测试，行为可验证
阶段五：多 Agent 并行执行 → Discord 指挥 + clawhip/0mX/0mO 协调
```

1~4阶段是准备。5阶段才是真正的coding。接下来我们逐一讲解。

## 2.5.2 五个阶段详解

### 2.5.2.1 归档 + 快照原始代码表面

简单来说你可以理解为就是去把整个项目做一个拆解，去将ClaudeCode代码变成可以量化的文字，比如有多少工具，有多少文件，有多少Commands（就是我们讲的在Claude里面输入/models,这种叫做command）。将大型项目变成快照，这样AI好查找，比如查找对应的某个command对应的代码是啥，做进度统计的时候也好去统计比如完成了哪些实现。这里面提到的.json文件，大家可以在后面提供的项目开源仓库中找到。

**第一步不是写代码，而是把要重写的东西完整记录下来，形成机器可查的"参考清单"。**

Claude Code 的代码组织非常规整——所有命令集中注册在 `src/commands.ts`，所有工具集中注册在 `src/tools.ts`，启动流程定义在 `src/entrypoints/cli.tsx`。每个命令/工具都是一行标准的 `import` 语句：

因为格式统一，用逐行字符串匹配就能提取出所有名字和路径——，不需要 AI 介入。

具体做了三件事：

1. **获取原始 TypeScript 源码快照。** 将 Claude Code 的原始 TS 源码存放在 `archive/claude_code_ts_snapshot/src`（在 `.gitignore` 里忽略，不公开上传），作为本地参考基准。

2. **compat-harness 自动提取清单。** 仓库中的 `rust/crates/compat-harness` 是一个 Rust crate，专门解析上游 TS 源码：扫描 `commands.ts` 逐行匹配 `import { xxx } from "./commands/..."`，提取出 207 个斜杠命令；扫描 `tools.ts` 提取 184 个工具模块；扫描 `cli.tsx` 搜索关键词提取启动流程的 7 个阶段。这是一个"活的"提取器——当上游 TS 代码更新时可重新运行，自动刷新清单。

3. **生成结构化快照 JSON。** 提取结果固化为四组 JSON 文件，存放在 `src/reference_data/` 目录：
   - `archive_surface_snapshot.json`：原始仓库的整体结构（18 个根文件、35 个子目录、1,902 个 TS 文件）
   - `commands_snapshot.json`：所有命令的 `name` + `source_hint` + `responsibility`（207 条）
   - `tools_snapshot.json`：所有工具模块的 `name` + `source_hint` + `responsibility`（184 条）
   - `subsystems/*.json`（29 个文件）：各子系统的模块数和样本文件名（assistant、bridge、utils 等）

局限性：这种字符串匹配只适用于代码结构规整的项目，如果用动态注册或插件扫描就行不通。

### 2.5.2.2 Python 镜像工作区

我个人的理解还是在建索引。上面是把Claude Code拆解了，拆解成json，让AI好查找。现在是把这个Claude Code变成一个python的骨架，这个骨架代码没实现，但是AI可以很快看出它的目录，它是如何启动的，整个流程是啥。相当于给了AI一个空壳，虽然没有实现，但是通过空壳可以快速知道项目结构，启动流程。

不直接从 TS 翻 Rust——直接翻译等于盲写。原始 TS 有 1,902 个文件、51 万行代码，AI agent 拿到工单后如果要自己在里面翻找架构信息，效率极低且容易遗漏。Claw Code 先建了一个 **Python 中间层**——一个只有 67 个文件的轻量"沙盘模型"，让 AI agent 在后续 Rust 重写时可以先查这个中间层快速定位，而不需要大海捞针。

这个 **Python 中间层**包含四样东西：

**1:1 结构映射表**。`src/parity_audit.py` 定义了 TS→Python 的完整映射关系，原始的 18 个根文件和 35 个子目录全部有对应的 Python 占位物。这张表的作用是让任何人（或 agent）一眼看清"原始系统有哪些组成部分，每个部分对应到 Python 层的哪个文件"。

**占位包**（29 个子目录的结构索引）。29 个子目录（`src/assistant/`、`src/bridge/`、`src/utils/` 等）各有一个 `init.py`，但不实现任何业务逻辑。每个占位包只做一件事——从 JSON 加载该子系统的元数据，导出 `MODULE_COUNT`（有多少模块）和 `SAMPLE_FILES`（包含哪些文件）。当 AI agent 收到"实现 assistant 模块"的工单时，读一下占位包就知道原始子系统有 1 个模块、文件叫 `sessionHistory.ts`，然后直接去对应的 TS 源码中精准阅读，而不需要在 1,902 个文件里盲目搜索。

**关键架构模型**（5 个 Python 文件复现核心决策逻辑）。光知道"有哪些模块"不够，还需要知道"系统怎么运转"。Python 层用 5 个文件把原始系统最关键的几条逻辑管线提取出来，编码为可运行的 Python 代码：

- `bootstrap_graph.py`：启动流程的 7 个阶段——prefetch → 环境检查 → CLI 解析 → 命令加载 → 延迟初始化 → 模式路由 → 查询主循环
- `command_graph.py`：把 207 个命令分类为 builtins（185）、plugin-like（20）、skill-like（2）
- `query_engine.py`：复现查询引擎的 turn-loop 行为——轮次限制（max turns）、token 预算执行（max budget）、消息压缩触发（compaction）。这里有真实的状态管理逻辑，不是 stub
- `setup.py`：建模 workspace 环境发现和启动 preflight 流程
- `runtime.py`：建模运行时路由——把用户的 prompt 分词、和所有命令/工具的名字做匹配打分、返回 top-N 结果。这也是真实的匹配算法，给一个 prompt 会返回实际的路由结果

**可运行的审核 CLI**。Python 层提供了 `python -m src.main`（40+ 个子命令），供人类和 AI agent 随时验证中间层的自治性：`summary` 输出工作区摘要，`parity-audit` 对比覆盖率（确保没有遗漏的模块），`route <prompt>` 验证路由逻辑是否和预期一致，`bootstrap <prompt>` 跑通完整的启动→路由→执行→历史记录流程。这个 CLI 同时也是 Rust 重写的行为基准：Rust 版的路由结果应该和 Python 版匹配，不匹配就说明实现有偏差。

### 2.5.2.3 量化重写进度

其实就是进度表。我们的rag项目其实也有进度表，但是没有他这么细致。它的进度表定义了文件覆盖率，目录覆盖率，目标覆盖率等，同时定义了AI如何开发，将整个项目开发分成9个可以并行的模块，这些设计需要你对Claude Code项目本身有一定的理解。里面的Parity.md就是进度表，大家在源码中可以看到。

核心思想：把"重写进度"从感觉变成机器可检查的量化指标。具体是通过三样东西实现的：一个自动审计脚本、两份 PARITY 清单文档、以及一个 9-lane 并行开发模型。

自动审计脚本。 `parity_audit.py` 中的 `run_parity_audit()` 自动扫描当前代码库，计算四个维度的覆盖率：根文件覆盖率（18 个中实现了多少）、目录覆盖率（35 个中覆盖了多少）、命令覆盖率（207 个中实现了多少）、工具覆盖率（184 个中实现了多少），并列出所有缺失的目标。测试中有断言确保覆盖率只升不降——写了一个新模块覆盖率上去了，后面不允许退化回来。

**PARITY.md 双层清单**。项目维护两份 PARITY 文档，分别追踪不同粒度的进度：
*   顶层 `PARITY.md`：记录 9 条开发 lane 的宏观状态——每条 lane 的 feature commit hash、merge commit hash、diff 行数统计，做到全程可追溯
*   `rust/PARITY.md`：更细粒度，对 40 个工具逐个标注实现深度，分四档：strong parity（行为完全对齐）、good parity（核心路径对齐）、moderate parity（基本可用）、stub（仅占位）

**9-lane 并行开发模型**。人类根据前两个阶段产出的清单，把重写工作切分为 9 条相互独立的 lane，每条一个 feature branch，可以由不同的 AI agent 并行推进，互不阻塞：
*   **Lane 1 — Bash validation**：9 个验证子模块（+1,005 行）
*   **Lane 2 — CI fix**：sandbox 探测修复（+22/-1 行）
*   **Lane 3 — File-tool**：边界检查——二进制检测、大小限制、路径穿越防护（+195/-1 行）
*   **Lane 4 — TaskRegistry**：内存任务生命周期管理（+336 行）
*   **Lane 5 — Task wiring**：6 个 task 工具接入 registry（+79/-35 行）
*   **Lane 6 — Team+Cron**：团队和定时任务注册表（+441/-37 行）
*   **Lane 7 — MCP lifecycle**：MCP 服务器连接、资源、工具分发桥接（+491/-24 行）
*   **Lane 8 — LSP client**：LSP 诊断、hover、定义、引用、补全（+461/-9 行）
*   **Lane 9 — Permission enforcement**：工具门禁、文件写入边界、bash 只读检查（+357 行）

每条 lane 独立 merge，互不阻塞。PARITY.md 中记录了每条 lane 的 feature commit hash 和 merge commit hash，做到全程可追溯。

### 2.5.2.4 确定性行为验证

> 虽然不同的厂商的 Harness 工程其实是有区别的，不过对于这一点，是统一的。我们在上面讲 Harness 工程的原则以及 Anthropic 公司的 Harness 的例子都涉及了。就是你不要让 AI 来判断是否完成，而是给出量化标准，脚本。所以这里需要设计一些让 AI 执行，评估的脚本。AI 通过这些标准和脚本来判断是否完成，记录到项目进度（2.5.2.3 讲的 parity.md）中。

重写不能靠"编译通过就算对"，但也不能每次测试都调真实 Anthropic API（贵、慢、结果不确定）。解法是建一套确定性的端到端测试体系，不调真实 API 也能验证 Rust 重写的行为是否和原始系统一致。这套体系由三样东西组成：一个 mock 服务、一组场景脚本、以及一个双向验证工具。

**Mock Anthropic 服务。** 这个场景脚本其实就是一些测试用例，进度文档需要根据这些测试用例来判断是否将某些模块标记为完成。

**10个场景脚本**。在 `mock_parity_scenarios.json` 中定义了 10 个测试场景，覆盖了核心工具链路、权限拒绝、多工具同轮、插件路径等关键行为。每个场景都通过 `parity_refs` 字段指向 `PARITY.md` 中的具体条目，确保测试和文档一一对应：

*   `streaming_text` (baseline)：纯文本流式响应，无工具调用
*   `read_file_roundtrip` (file-tools)：文件读取往返
*   `grep_chunk_assembly` (file-tools)：grep 分块 JSON 组装
*   `write_file_allowed` (file-tools)：workspace-write 模式下写入成功
*   `write_file_denied` (permissions)：read-only 模式下写入被拒绝
*   `multi_tool_turn_roundtrip` (multi-tool)：同一 turn 内执行多个工具
*   `bash_stdout_roundtrip` (bash)：bash 执行和 stdout 往返
*   `bash_permission_prompt_approved` (permissions)：bash 权限提示 → 批准
*   `bash_permission_prompt_denied` (permissions)：bash 权限提示 → 拒绝
*   `plugin_tool_roundtrip` (plugin)：加载外部插件工具并执行

我们做rag项目的时候，就有进度文档和实际代码可能不一致的问题，这里他给出了解决方案。用专门的脚本来判断二者是否发生偏移，发生偏移了即时纠正。
**双向验证工具（防止文档和代码漂移）。** `run_mock_parity_diff.py` 做两件事：先遍历每个场景脚本的 `parity_refs`，确认每条引用在 `PARITY.md` 中**确实存在**（找不到就报错）；再运行 `cargo test` 输出每个场景的通过状态。这意味着 `PARITY.md` 不只是给人看的文档——它是场景清单的锚点，文档改了但测试没更新、或者测试加了场景但文档没跟进，都会被这个工具检测到。

### 2.5.2.5 多Agent协作执行
这一步才是真正的执行。执行也是很有技巧，当时我们的rag项目都是串行的，一个agent。但是人家是并行，多个Agent。甚至专门写了3个工具负责Agent调度，执行，监控。（这三个工具也给出了代码仓库，大家想深入学习可以学习）。Anthropic公司的harness项目也是用的多Agent。大佬们做的Harness工程，还是得用多Agent来做。所以自己做Harness的时候，最好用多Agent，想想他们如何调度，监测，执行的，这样面试时候也会显得更高级。

如 `PHILOSOPHY.md` 所述，这个项目的核心哲学是：人类提供方向，`claws`（AI agent）执行劳动。前四个阶段建好了清单、中间层、进度条和测试题，第五阶段就是让多个 **AI agent** 并行干活，人类只在关键节点介入。

为了实现这一点，项目作者编写了三个专用编排工具（不是通用框架，而是为这套 Vibe Coding 工作流定制的）：

*   **OmX** (`oh-my-codex`) ——任务拆解器：把一句话指令转化为结构化工单，定义子任务、执行模式和验证标准（仓库地址：https://github.com/Yeachan-Heo/oh-my-codex）
*   **clawhip** ——后台监控员：监听 git commits、CI 结果、PR 状态、agent 生命周期事件，在 agent 上下文窗口之外独立运行，只推送通知不干扰执行（仓库地址：https://github.com/Yeachan-Heo/clawhip）
*   **OmO** (`oh-my-openagent`) ——角色协调员：管理 Architect/Executor/Reviewer 之间的分工和分歧解决，确保多 agent 循环收敛而不是无限争吵。（仓库地址：https://github.com/code-yeongyu/oh-my-openagent）

有了这三个工具，加上前四个阶段建好的清单和测试，就能实现"人类发指令、agent 自主干活"的工作流。完整流程如下：

```
人类发指令 → OmX 拆任务 → OmO 分角色/启动 agent
→ agent 自主循环（写码→测试→修复）
→ clawhip 后台监控推送事件
→ 人类只在失败或完成时介入判断
```

1. 人类输入一句高层指令（通过 Vibe Coding 工具），例如："实现 Lane 3：给 file tool 加边界检查"
2. OmX 自动拆解成子任务（二进制检测、大小限制、路径穿越防护），确定执行模式（并行/串行）和验证标准（cargo test + mock parity 通过）
3. OmO 分配角色并启动 agent：Architect 规划方案、Executor 写代码、Reviewer 审查质量。多组 agent 在服务器上并行工作
4. Agent 自主循环：写代码 → 跑测试 → 失败 → 读错误 → 改代码 → 再跑测试。Reviewer 不满意就打回重做，OmO 协调分歧直到收敛。**全程不需要人类参与**
5. clawhip 在后台监控所有事件（`lane.started`、`lane.commit.created`、`lane.red`、`lane.green`、`lane.pr.opened`、`lane.finished` 等），向人类推送状态通知，但不打扰正在干活的 agent——agent 的上下文窗口是宝贵资源，不能被监控噪音占据
6. 人类只在两种情况介入：agent 自己恢复不了的失败（决定重试还是换方案），或 lane 完成后判断能否 merge

### 2.5.3 设计原则

这套方法论的核心是四条设计原则：

1. **人定标准，AI 执行。** 所有判断标准由人预先制定——覆盖率数字、parity 标签体系（strong/good/moderate/stub）、mock 场景清单。AI agent 不需要自己判断"够不够好"，只需对照人制定的验收标准逐项打勾。
2. **建中间层，降低 AI 认知成本。** 51 万行 TS 太大，AI 无法每次从头阅读。用 JSON 索引 + Python 骨架建一个轻量参考层，让 agent 查一下就能定位和理解，而不需要大海捞针。
3. **用工具编排多 agent，而非人工协调。** Python 脚本生成"做什么"（清单），OmX/clawhip/OmO 三件套负责"谁来做、怎么分发、怎么验收"，人类不需要坐在终端前逐个分配任务。
4. **人只做方向、分解、判断。** 人基于架构理解把工作拆成 9 条独立 lane，定义 lane 边界和合入标准。292 个 commit 中，人的贡献集中在：重写什么（方向）、怎么拆（分解）、何时 merge（判断）。剩下的全是 agent 的事。

### 2.5.3 项目地址

> 这个项目本身是个很好的学习Harness技巧的项目，你可以通过这个项目，学习设计Harness，自己通过他的方法，复现，自己也可以实现一个Claude Code的Agent。这是很值得做的事。深入的学习靠大家自己了。我后面想要在我自己做的Agent项目再设计一个Harness工程，我就不会深入研究这个项目了。如果你不想深入学习，最少通过这些笔记，你能更进一步理解Harness项目的设计思想，也能够在面试的时候有更多可说的。

https://github.com/seavee/ClawCode

## 2.5.4 讲解视频
我用夸克网盘给你分享了「Clawcode 5层Harness架构.mp4」，点击链接或复制整段内容，打开「夸克APP」即可获取。

/-3e793Y82Xv-:/

链接：https://pan.quark.cn/s/d145e110cc10

## 2.6 面试问题速览
（刚才上面的文档是给大家理解学习用的，整个章节的问题是供大家面试时直接照着回答的）

### 2.6.1 什么是 Harness Engineering?
Harness Engineering 是围绕 AI 模型构建运行环境、调度机制和管控基础设施的一种工程实践。它的目标不是单纯让模型"更聪明"，而是让 AI Agent 在长时、复杂、多步骤的真实任务中，能够稳定、可靠、可控地持续运行。
如果用一句更通俗的话来说，Harness Engineering 做的事情，就是不再把 AI 当成一个一次性回答问题的黑盒，而是把它放进一个被设计好的工作系统里，让它按规则做事。
它背后的设计哲学也很重要：模型只是原材料，环境才决定结果能不能落地。
再强的模型，如果直接裸跑长任务，也会遇到上下文腐化、失忆、幻觉完成、任务漂移等问题。所以 Harness Engineering 的核心，不是继续往模型上堆能力，而是通过外部记忆、任务拆解、测试门禁、状态交接和人工审核，把 AI 的执行过程工程化。

### 2.6.2 Harness Engineering 想解决的核心问题是什么？
它主要解决的是 AI 在长时任务里的可靠性问题。
第一个问题是 Context Rot，也就是上下文腐化。模型做长任务时，上下文窗口会不断被历史对话、错误尝试和无关信息填满，最后导致它丧失对原始目标的把握。
第二个问题是 Hallucinated Completion，也就是幻觉完成。模型在复杂任务中迷失后，往往不会老实说"我不会了"，而是会输出一个看起来完成、实际上并不正确的结果。
第三个问题是 Model Drift，也就是模型漂移。多步执行中，模型会逐渐偏离最初目标，最后做出来的东西方向就歪了。
所以我理解 Harness Engineering 的价值，不是让模型在单轮回答里更强，而是让它在几十步、几百步的连续工作流里，依然能沿着正确轨道往前走。

### 2.6.3 Harness Engineering 和 Prompt Engineering、Context Engineering 的关系是什么？
我会把它理解成一个演进关系，而不是替代关系。
Prompt Engineering 主要解决的是"对 AI 说什么"，也就是如何设计指令，让单次输出更好。
Context Engineering 进一步解决的是"让 AI 知道什么"，也就是在一次 session 或上下文窗口里，给它什么信息、给多少信息、怎么组织信息。
而 Harness Engineering 更进一步，它解决的是"让 AI 在什么环境里做事"。

也就是说，Harness Engineering 并没有取代前两者，而是把它们包进了一个更大的系统里。Harness 内部的每个 session 还是要做 Context Engineering，每一次给 agent 的具体任务描述还是要 Prompt Engineering。只是现在关注点从"优化一次对话"升级成了"设计一整个运行体系"。

### 2.6.4 Harness Engineering 的最佳实践是什么？
我会总结成几个比较关键的实践原则。
第一，状态外部化。不要把连续性寄托在模型记不记得，而要把状态写进文件、Git 或数据库，让系统本身有记忆。
第二，任务原子化。不要让一个 agent 一口气做一个大需求，而是拆成足够小、足够明确的原子任务，每次只做一件事。
第三，上下文刷新。很多先进的 harness 架构都会在一个小任务完成后销毁当前 agent，再启动一个全新的 agent 来继续做下一轮。这么做的目的，就是从根本上避免上下文腐化。
第四，验证优先。AI 的输出必须经过测试和规则检查，而不是靠它自我声明"完成了"。这是把模型主观判断变成工程确定性的关键。
第五，模型无关性。Harness 不应该绑定特定模型，新模型出来能快速替换、快速验证。
第六，Harness 本身也是数据集。每次 agent 失败、漂移、异常都是宝贵的训练素材，用来迭代优化 harness 本身。
第七，Harness 可拆可替换。模块要为删除而建，能够快速移除替换，而不是过度编码人类知识。Manus 6个月5次重构、Vercel 删 80%手工工具反而更好，就是这个原则的体现。
第八，Harness 应该在没有人工干预的情况下替换模型、替换技能、替换验证逻辑。

### 2.6.5 如果让你用Harness工程来实现一个项目，你怎么设计？

（套用我们上面学的Anthropic公司的方案，也可以在哪个问题中通过整个回答介绍Anthropic公司的Harness方案，灵活运用，学了就不浪费）

如果让我来落地一个 Harness，从一个 Initializer + Coding Loop 的最小闭环开始，而不是一开始就做复杂的多 agent 平台。

第一步，我会先准备一份清晰的 App Spec / PRD，把产品目标、功能范围和验收标准定义清楚。因为在 Anthropic 的架构里，系统起点不是直接写代码，而是先把需求整理成后续 agent 可以反复执行的任务蓝图。

第二步，我会实现一个 Initializer Agent。它的职责不是写业务代码，而是读取需求文档，把大任务拆成一组细粒度、可执行、可验证的 feature，并生成类似 feature_list.json 这样的状态文件。同时它还会初始化项目骨架、代码仓库，以及后续循环执行所需的基础环境。这样系统就有了"任务清单"和"状态来源"。

第三步，我会实现一个 Coding Loop。每一轮都启动一个全新的 coding agent，让它只读取当前状态文件、进度记录和代码库信息，只完成一个明确的小任务。任务完成后，不是让 agent 自己宣布成功，而是必须经过测试、lint 或类型检查等确定性验证；只有验证通过，才更新状态文件、写入交接记录并提交代码。然后这个 agent 立即退出，下一轮由新的 agent 接力。

第四步，我会把外部记忆做成系统核心，而不是把连续性寄托在模型上下文里。也就是说，我会依赖 feature list、progress file、Git log 这些可持久化信息，让每个新 agent 启动时都能快速接手，而不是依赖前一轮 session 的"记忆"。

第五步，我会在关键节点加入 Human-in-the-loop。Anthropic 的方法虽然高度自动化，但并不是完全无人值守。所以在高风险功能、关键界面变更或者多轮失败之后，我会设计人工确认断点，确保系统不会沿着错误方向持续推进。

整体上，我不会把重点放在"怎么让一个模型一次做完所有事"，而是把重点放在：先拆解、再循环执行、靠外部状态保持连续性、靠确定性验证保证质量。我认为这是一种更可落地、也更适合真实工程环境的 Harness 实现方式。

### 2.6.6 你看过哪些 Harness 工程的实际案例，可以分享一下吗？

我比较熟悉两个案例，一个是 Anthropic 官方的 Harness 架构，一个是开源社区的 Claw Code 项目。它们一个是从零做新项目的范式，一个是重写已有大型系统的范式，刚好互补。

第一个是 **Anthropic 官方的 Harness 架构**。这是一个从项目文档出发做新项目的标准流程。核心思路是"每个 agent 只活一次，干完就死"：

先由 Initializer 读需求文档，拆成 feature list 和验收标准，初始化项目骨架；然后进入 Coding Loop，每轮启动一个全新的 agent，它只读当前状态文件和代码库，完成一个原子任务，通过测试后更新状态文件并退出，下一轮由新 agent 接力。项目记忆全部存在外部文件（feature list、progress file、Git），不依赖任何一个 agent 的上下文窗口。验证也不靠模型自己说"完成了"，而是必须通过测试、lint、类型检查这些确定性手段。

这个架构最核心的 insight 是：用"短命 agent 接力"代替"长命 agent 硬撑"，从根本上避免上下文腐化。

第二个是 **Claw Code 项目**。这是 2026 年 3 月 Claude Code 源码泄漏后，开源社区用多个 AI agent 在 3 天内把核心功能从 TypeScript 重写为 Rust 的一个实践。它和 Anthropic 方案的最大区别在于：它不是从零写新项目，而是重写一个已有的 51 万行系统。

它的 Initializer 阶段不是读项目文档拆 feature，而是先扫描原始代码，把结构提取为机器可查的 JSON 清单，再建了一个 Python 中间层作为 AI agent 的"架构速查手册"，降低 agent 理解已有系统的认知成本。验证方式也不同——因为是"重写"，所以需要验证行为一致性，它建了一个 mock API 服务做确定性端到端测试，还做了文档和测试的双向锚定防止漂移。编排上用了三个定制工具分别负责任务拆解、后台监控和角色协调，人类只负责方向、分解和 merge 判断。

对比来看：两个案例的底层原则一致——状态外部化、任务原子化、确定性验证、人只做关键决策。但适用场景不同：Anthropic 方案更适合从零开始的新项目，核心 insight 是"agent 用完即销毁"避免上下文腐化；Claw Code 更适合代码迁移和重写场景，核心 insight 是"建中间层降低 agent 认知成本"应对已有代码库的复杂性。
