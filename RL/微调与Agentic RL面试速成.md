# 微调与 Agentic RL 面试速成指南

> 定位：加分项知识，面试中能聊 5-10 分钟即可。重点是理解脉络和关键概念，不需要深入数学推导。

---

## 1. LLM 训练的三阶段全景图

```
Pre-training (预训练)
    ↓  海量文本，next token prediction
SFT (监督微调)
    ↓  高质量 instruction-response 对
RLHF / RL (强化学习对齐)
    ↓  让模型输出更符合人类偏好 / 更正确
```

面试核心认知：SFT 教模型"怎么说话"，RL 教模型"说得更好"。

---

## 2. RLHF 经典流程（必须能讲清楚）

### 三步走：

1. **收集偏好数据**：同一个 prompt，模型生成多个回答，人类标注哪个更好（A > B）
2. **训练 Reward Model (RM)**：用 Bradley-Terry 模型学习人类偏好，输出一个标量分数
3. **PPO 优化策略模型**：用 RM 的分数作为奖励信号，通过 PPO 算法更新 LLM 权重

### PPO 的关键组件（4 个模型）：

| 模型 | 作用 |
|------|------|
| Policy Model (Actor) | 当前正在训练的 LLM |
| Reference Model | 冻结的原始 LLM，用于计算 KL 散度防止偏离太远 |
| Reward Model | 给生成结果打分 |
| Value Model (Critic) | 估计状态价值，计算 advantage |

PPO 的核心公式思想：最大化奖励的同时，用 KL 惩罚约束策略不要偏离 reference model 太远。

### PPO 的痛点：
- 需要同时维护 4 个模型，显存开销巨大
- 训练不稳定，超参敏感
- Reward Model 本身可能有偏差（reward hacking）

---

## 3. DPO：去掉 Reward Model 的捷径

DPO (Direct Preference Optimization) 的核心洞察：

> 可以把 RM 训练 + PPO 优化这两步合并成一步，直接用偏好数据优化策略模型。

```
RLHF:  偏好数据 → 训练 RM → PPO 优化 LLM
DPO:   偏好数据 → 直接优化 LLM（隐式学习奖励）
```

DPO 损失函数的直觉：让模型对"好回答"的概率相对于"差回答"的概率更高，同时用 reference model 做正则化。

优点：简单、稳定、省资源
缺点：离线学习，不能在线探索；对数据质量要求高

---

## 4. GRPO：DeepSeek R1 的核心算法（2025 热点，必问）

GRPO (Group Relative Policy Optimization) 是 DeepSeek 提出的，用于训练 R1 推理模型。

### 核心思想：

对同一个 prompt，采样一组（group）回答（比如 G=64 个），然后：
1. 用 reward function 给每个回答打分
2. 在组内计算相对优势（advantage）：`A_i = (r_i - mean(r)) / std(r)`
3. 用这个 group-normalized advantage 更新策略

### 相比 PPO 的关键改进：

| | PPO | GRPO |
|---|---|---|
| Value Model | 需要（Critic） | **不需要**，用组内均值替代 baseline |
| Reward Model | 需要训练 | 可以用 **规则验证**（如数学答案对不对） |
| 模型数量 | 4 个 | 2 个（Policy + Reference） |
| 显存 | 巨大 | 大幅减少 |

### DeepSeek R1 的训练流程（面试高频）：

```
阶段 1: R1-Zero（纯 RL，无 SFT）
  DeepSeek-V3-Base → GRPO 训练 → R1-Zero
  奖励：数学正确性 + 格式规范
  发现：模型自发学会了 chain-of-thought，出现 "aha moment"

阶段 2: R1（完整流程）
  V3-Base → Cold Start SFT（少量高质量 CoT 数据）
       → RL 阶段 1（推理任务 GRPO）
       → Rejection Sampling 生成 SFT 数据
       → SFT 阶段 2（混合推理 + 通用数据）
       → RL 阶段 2（全场景对齐）
```

### 面试金句：
- "R1-Zero 证明了纯 RL 就能涌现推理能力，不需要 SFT 作为前置"
- "GRPO 去掉了 Critic 模型，用组内相对排名代替绝对价值估计，训练更稳定也更省资源"
- "但 R1-Zero 有可读性差、语言混杂等问题，所以完整 R1 还是加了 SFT cold start"

---

## 5. RLVR：可验证奖励的 RL（新范式）

RLVR (Reinforcement Learning with Verifiable Rewards) 是 2025 年的重要趋势。

核心区别：
```
RLHF:  奖励来自人类偏好（主观）→ 需要训练 Reward Model
RLVR:  奖励来自规则验证（客观）→ 直接用程序判断对错
```

适用场景：
- 数学：答案是否正确
- 代码：能否通过测试用例
- 逻辑推理：结论是否成立

GRPO + RLVR 就是 DeepSeek R1 的训练范式。

---

## 6. Agentic RL：让模型学会使用工具（你的重点关联）

这是微调 RL 和 Agent 的交叉点，面试中可以自然地把话题引到你的 Agent 经验上。

### 什么是 Agentic RL？

传统 RL 微调：模型生成文本 → 获得奖励
Agentic RL：模型生成文本 + **调用工具** + 观察结果 + 继续推理 → 获得奖励

```
Agent 的一个 episode:
  Prompt → Think → Tool Call → Observation → Think → Tool Call → ... → Final Answer
                                                                         ↑
                                                                    奖励信号在这里
```

### 关键挑战：

1. **长 horizon 信用分配**：最终奖励要分配给中间的每一步决策（哪次 tool call 是关键的？）
2. **工具交互的不确定性**：工具返回结果是动态的，不像纯文本生成那样可控
3. **探索空间爆炸**：工具组合 × 参数组合 × 推理路径，搜索空间巨大
4. **奖励设计**：如何定义"好的工具使用"？最终结果正确？步骤最少？

### 代表性工作：

| 项目 | 要点 |
|------|------|
| **OpenAI Agent RFT** | 训练时模型真实调用工具端点，用 grader 提供奖励信号，模型学习最优工具调用策略 |
| **Agent-R1** | 模块化 RL 框架，扩展 MDP 加入工具调用转移，支持 dense process reward |
| **ToolBrain** | 轻量框架，支持 GRPO/DPO 训练 agent 的工具使用，支持 LLM-as-judge 做奖励 |
| **Multi-Agent RFT** | 多 agent 协作场景下的 RL 微调，用 MARL 建模 agent 间协作 |

### 面试中怎么聊：

> "Agentic RL 本质上是把 agent 的多轮工具调用过程建模为一个 MDP，用 RL 来优化整个 trajectory。
> 和我做的 agentic RAG 其实是一个思路——我的 agent 也是多轮决策（检索、分析、回答），
> 区别在于我是用 prompt engineering 来引导决策，而 agentic RL 是直接用梯度来优化决策策略。
> 如果要进一步提升我的系统，用 RL 微调 agent 的检索决策和工具选择是一个很自然的方向。"

---

## 7. 技术演进脉络（一张图记住）

```
RLHF + PPO (2022, InstructGPT)
  ↓ 太复杂，4个模型
DPO (2023)
  ↓ 去掉 RM，直接优化，但是离线的
GRPO + RLVR (2025, DeepSeek R1)
  ↓ 去掉 Critic，用可验证奖励，在线 RL
Agentic RL (2025-2026)
  ↓ RL 扩展到工具使用和多轮交互场景
Multi-Agent RL (前沿)
  ↓ 多个 agent 协作的 RL 训练
```

---

## 8. 常见面试问题 & 参考回答

### Q: RLHF 和 DPO 的区别？
A: RLHF 是两阶段的——先训练 RM 再用 PPO 优化；DPO 把这两步合成一步，直接用偏好数据优化策略。DPO 更简单稳定但是离线的，RLHF 可以在线探索但训练复杂。

### Q: GRPO 为什么不需要 Critic？
A: GRPO 对同一个 prompt 采样一组回答，用组内均值和标准差做归一化来计算 advantage，相当于用组内统计量替代了 Critic 的价值估计。这样少维护一个模型，训练更稳定。

### Q: DeepSeek R1 的 "aha moment" 是什么？
A: R1-Zero 在纯 RL 训练过程中，模型自发学会了在 CoT 中进行自我反思和纠错，比如输出 "wait, let me reconsider"。这说明推理能力可以通过 RL 涌现，不一定需要 SFT 来教。

### Q: 什么是 reward hacking？怎么缓解？
A: 模型找到了 RM 的漏洞，生成的回答 RM 打高分但实际质量差。缓解方法：KL 惩罚约束策略不偏离太远；用 RLVR 替代学习的 RM（规则验证不可 hack）；定期更新 RM。

### Q: Agentic RL 和普通 RLHF 的区别？
A: 普通 RLHF 是单轮生成文本获得奖励；Agentic RL 是多轮交互——模型可以调用工具、观察结果、继续推理，整个 trajectory 结束后获得奖励。核心挑战是长 horizon 的信用分配和工具交互的探索空间。

### Q: 如果让你用 RL 优化你的 Agent 系统，你会怎么做？
A: （结合你自己的项目回答）
1. 定义 reward：最终回答的准确性 + 检索效率（步骤数）
2. 把 agent 的决策过程建模为 MDP：state = 当前上下文，action = 选择工具/生成回答
3. 用 GRPO 训练，每个 query 采样多条 trajectory，用结果正确性做可验证奖励
4. 可以加 process reward 给中间步骤也提供信号，缓解稀疏奖励问题

---

## 9. 工具和框架（知道名字就行）

| 框架 | 特点 |
|------|------|
| **TRL (HuggingFace)** | 最主流，支持 SFT/DPO/PPO/GRPO，和 transformers 生态无缝集成 |
| **OpenRLHF** | 基于 Ray 的分布式 RLHF 框架，支持 70B+ 模型训练，支持 Async Agentic RL |
| **veRL (字节)** | 高性能 RL 训练框架，支持 FSDP/Megatron 后端 |
| **Unsloth** | 高效微调，QLoRA 加速，适合小团队 |

---

## 10. 面试策略建议

1. **主动引导**：聊到微调 RL 时，自然过渡到 "这和我做的 agent 系统其实是相通的..."
2. **深度适可而止**：能讲清 RLHF → DPO → GRPO 的演进脉络就够了，不需要推公式
3. **关联实际**：强调你理解这些技术如何应用到 agent 场景（agentic RL）
4. **诚实边界**：如果问到训练细节（如 GRPO 的具体梯度更新），可以说 "我更多是从应用层面理解，实际训练经验有限"

---

*参考来源：[DeepSeek R1 训练解析](https://www.philschmid.de/deepseek-r1)、[HuggingFace TRL GRPO Trainer](https://huggingface.co/docs/trl/main/en/grpo_trainer)、[Agentic RL with Tool Integration](https://arxiv.org/html/2505.01441v1)、[OpenAI Agent RFT](https://www.startuphub.ai/ai-news/ai-video/2025/openais-agent-rft-boosting-autonomous-ai-performance-through-tailored-reinforcement-learning)*


---

## 11. 各概念层级关系总览

> 以下为速查总览。训练基础（Loss、梯度、AdamW、LoRA 等）、DPO 隐式奖励、在线 vs 离线、PPO/DPO 训练节奏等深度内容，详见《从零学 SFT 与 RL》。

```
RL（学习范式，和监督学习并列）
 │
 ├── RLHF（应用框架：奖励来自人类偏好）
 │    ├── PPO 路线（显式 RM + 在线 RL，4 个模型）
 │    ├── DPO 路线（隐式 RM + 离线优化，2 个模型）
 │    └── 其他变体：KTO / IPO / ORPO / SimPO
 │
 ├── RLVR（应用框架：奖励来自规则验证）
 │    └── GRPO（去掉 Critic，组内相对排名，2 个模型）
 │
 └── Agentic RL（应用框架：RL 用于多步工具调用）
      └── Agent RFT / Agent-R1 / ToolBrain 等

LoRA / QLoRA → 正交的参数效率技术，可与上述任意范式组合
```

### 面试中怎么讲这个层级

> "RL 是大的学习范式，RLHF 是它在 LLM 对齐中的应用框架，定义了'奖励从哪来'——从人类偏好来。PPO 和 DPO 是 RLHF 框架下两条不同的技术路线：PPO 走显式奖励模型 + 在线 RL，DPO 走隐式奖励 + 离线优化。LoRA/QLoRA 是正交的效率技术，跟训练范式无关，可以和任何一种组合使用。"


---

*DPO/PPO 深度对比（在线 vs 离线、训练循环、偏好数据收集、Loss 计算等）详见《从零学 SFT 与 RL》。*
