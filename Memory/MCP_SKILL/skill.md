
---

# 3 SKILL

SKILL 比较新，是2025年10月16日提出，在12月份左右才慢慢被市场支持，在26年1月份已经变得很火了。所以虽然目前我的面经没有涉及，但是很多朋友在1月份面试已经被问到了，而且这个概念现在很火，学习大模型，特别热门的新技术一定是要学的。从目前（2026年1月）来看，Skills会成为未来一段时间面试高频考点，务必要掌握。推荐学习路线为：

1. 入门视频：了解Skill是什么，怎么用。（视频见3.10）
2. 进阶视频：掌握Skill的设计思路。（视频见3.10）
3. **Skill编写规范**：本节的笔记部分以及我在下面推荐的Anthropic关于Skill的官方文档：了解Skill编写的最佳实践，范式，命名规则等。
4. **如何写SKILL**：下面推荐视频有个博主讲如何写出SKILL的，以及现场教学。最重要的思想：**Feedback Cycle**其实就是反复迭代，了解了1，2，3，可以看一下他展示真的如何写的这个思想和过程。
5. 实战：
   1. 此节笔记提供了一些非官方的SKILL github链接，特别是第4个链接，是由Anthropic提供的Skill示例，严格遵循了它提供的Skill的最佳实践文档，可以作为Skill的教科书级别的示例来阅读。（链接见9.10）
   2. 我自己也录制了一个视频，结合我们笔记中的Rag项目对应的SKILL来讲解我写SKILL遇到的坑，怎么写出来的Skill，怎么使用Skill-creator之类的问题给大家参考。
6. 最后是你应该自己写一些SKILL？自己用的提效的SKILL，甚至是公司用的更标准的SKILL，写一些你就会更有经验。

通过这个路线，你的SKILL技巧应该比较成熟了。面试基本SKILL会问：是什么？怎么用？你用它做了什么事？你写了什么SKILL？相信你通过这个路线学了，这几个问题都能答出来。

### 3.1 SKILL 概念

Skill是一组打包了指令的文件夹：文件夹会包含Skill.md的核心指令文件，以及Scripts,Reference和assets的可选资源文件（后文SKILL目录结构小节有更详细解释）。它教会AI如何处理制定的任务和流程。当需要处理重复的工作流，比如根据规范生成前端设计，创建符合团队风格指南的文档，多步骤流程编排等任务时，Skill可以让你无需每次对话重新解释你的偏好，流程，领域知识，而是通过Skill一次教会AI，之后每次都能受用。相比传统的提示词，它采用渐进式纸漏，动态加载提示词，更节约Token。

### 3.2 SKILL 目录结构

从图中我们可以清晰的理解概念所说，一个SKILL就是一个文件夹。其中包含以下内容：

```
your-skill-name/
├── SKILL.md                # Required - main skill file
├── scripts/                # Optional - executable code
│   ├── process_data.py # Example
│   └── validate.sh # Example
├── references/             # Optional - documentation
│   ├── api-guide.md # Example
│   └── examples/ # Example
└── assets/                 # Optional - templates, etc.
    └── report-template.md # Example
```

1.SKILL.md：必须，包含了 Yaml头和Body。Yaml是md文件中的头，告诉AI是否加载这个SKILL，AI必定会加载。而body是正文，是渐进式披露的，只有AI读取了YAML头，认为这个SKILL应该被加载，才会加载这个正文部分。

2.Scripts：可选。包含着在AI执行过程中需要使用的脚本文件。
3.References：可选。存放执行SKILL相关的文档，说明，内容以MD为主。
4.Assets：可选。存放执行SKILL是需要的模板，静态资源，比如图片等。

2,3,4其实就是在SKILL执行过程中，所需要的资源。Anthropics 官方明确建议一个SKILL不应该超过500行，所以SKILL需要引用其它资源的时候，应该将这些东西放在SKILL.MD之外，而在SKILL.MD中显示指向它。

## 3.3 SKILL的常见应用场景：
1.生成一致的，高质量的内容，比如文档，PPT，app，design，代码等。实例：[fronted-design skill](fronted-design skill)
2.编排多步流程，让这些步骤遵循特定的顺序，示例：[Skill-creator](Skill-creator)
3.规范/增强MCP使用，将MCP转换成可靠的工作流。实例：[Sentry-code-review skill](Sentry-code-review skill)
（这一点我想解释一下的是，根据Anthropics官方指南，规范MCP的使用指的是用了Skill, 可以让用户/AI更清楚知道某个MCP Server 如何使用，遇到问题如何解决，取得稳定的结果。通常的最佳实践是在MCP Server的文档中附上SKILL的链接。用户在使用的多个MCP Server的时候，也可以通过SKILL来编排多个MCP的使用流。从这一点可以看出SKILL和MCP是相辅相成的，而不是替代关系。这一点是Anthropics官方指南，我觉得这一点比较新颖，也是一些视频没有提到的，面试时候说出一点SKILL和MCP的补充关系和最佳实践，可以加分）

## 3.4 SKILL测试与评估
通常可以从以下几个角度来测试Skill的质量：
a. 触发测试：保证SKILL会在正确的时机被加载。在不该触发的情况不会被误触发，在该触发的情况也不会不触发。
b. 功能测试：测试SKILL产生正确的结果：输出是符合预期的，API正确调用，错误被处理，边缘情况被覆盖。
c. 性能测试：对比使用SKILL和不适用SKILL时，用户的介入次数是否减少，API调用失败次数是否减少，Token消耗是否降低。

## 3.5 SKILL典型范式

a. 顺序工作流编排：当有多个步骤，并且需要按照指定的顺序执行时使用。
核心要点：说明步骤顺序，步骤间的依赖，每一步如何验证，失败时如何回滚。

```markdown
## Workflow: Onboard New Customer

### Step 1: Create Account
Call MCP tool: `create_customer`
Parameters: name, email, company

### Step 2: Setup Payment
Call MCP tool: `setup_payment_method`
Wait for: payment method verification

### Step 3: Create Subscription
Call MCP tool: `create_subscription`
Parameters: plan_id, customer_id (from Step 1)

### Step 4: Send Welcome Email
Call MCP tool: `send_email`
Template: welcome_email_template
```

b. 多MCP协作：任务流包含多个MCP Server时使用。
核心要点：说明多MCP步骤间如何传递数据，错误异常处理，清晰的步骤界限；

```markdown
### Phase 1: Design Export (Figma MCP)
1. Export design assets from Figma
2. Generate design specifications
3. Create asset manifest

### Phase 2: Asset Storage (Drive MCP)
1. Create project folder in Drive
2. Upload all assets
3. Generate shareable links

### Phase 3: Task Creation (Linear MCP)
1. Create development tasks
2. Attach asset links to tasks
3. Assign to engineering team

### Phase 4: Notification (Slack MCP)
1. Post handoff summary to #engineering
2. Include asset links and task references
```

c. 迭代优化：需要通过多次迭代提升输出质量时：
核心要点：清晰的质量标准，迭代提升，验证脚本，知道何时停止迭代。

## Iterative Report Creation

### Initial Draft
1. Fetch data via MCP
2. Generate first draft report
3. Save to temporary file

### Quality Check
1. Run validation script: `scripts/check_report.py`
2. Identify issues:
   - Missing sections
   - Inconsistent formatting
   - Data validation errors

### Refinement Loop
1. Address each identified issue
2. Regenerate affected sections
3. Re-validate
4. Repeat until quality threshold met

### Finalization
1. Apply final formatting
2. Generate summary
3. Save final version

d. 上下文感知工具选择：当需要根据上下文场景，来动态选择使用不同的工具时使用
核心要点：清晰的选择策略，兜底选择，选择理由透明

---

## Smart File Storage

#### Decision Tree
1. Check file type and size
2. Determine best storage location:
   - Large files (>10MB): Use cloud storage MCP
   - Collaborative docs: Use Notion/Docs MCP
   - Code files: Use GitHub MCP
   - Temporary files: Use local storage

#### Execute Storage
Based on decision:
- Call appropriate MCP tool
- Apply service-specific metadata
- Generate access link

#### Provide Context to User
Explain why that storage was chosen

---

e. 领域专业智能：Skill需要使用特定领域的知识来完成任务时使用。
关键要点：将专业知识嵌入逻辑，合规性保证，详尽的领域文档，

---

## Payment Processing with Compliance

#### Before Processing (Compliance Check)
1. Fetch transaction details via MCP
2. Apply compliance rules:
   - Check sanctions lists
   - Verify jurisdiction allowances
   - Assess risk level
3. Document compliance decision

#### Processing
IF compliance passed:
   - Call payment processing MCP tool
   - Apply appropriate fraud checks
   - Process transaction
ELSE:
   - Flag for review
   - Create compliance case

#### Audit Trail
- Log all compliance checks
- Record processing decisions
- Generate audit report

---

## 3.6 SKILL vs MCP
(面试官特喜欢问的是差别，你答的不多，还喜欢追问，还有吗？这里尽量给大家总结全面一点)
借用Anthropic官网文档原话，Skills 和 MCP的核心差别是：MCP 提供给模型数据，而Skills教会模型如何使用这些数据（"MCP connects Claude to data, Skills teach Claude what to do with data"）。二者的区别为：

a. MCP侧重于提供工具调用，而Skills则侧重于提示词。MCP更像是一个标准的工具箱，提供规范，可复用的工具。而Skills则是一个带目录的说明书。

b. MCP的本质是一个独立运行的程序，而Skills本质是一个文档。他们本质不同，决定了他们性质也不同，Skills更适合处理较轻量级的，简单的逻辑，安全性和稳定性都不及MCP。

c. Skills 采用渐进式披露提示词的机制，Tokens消耗低。而MCP则是一次性塞入所有提示词，Token消耗高。

d. Skills的开发难度相比于MCP，开发难度低很多。

总结来说，SKILL和MCP二者不是相互替代，而是相互补充的。在开发MCP SERVER的时候，可以在MCP Server的文档中添加一个链接，指向一个SKILL的链接，这个SKILL让用户/AI更清楚知道该MCP如何使用，遇到问题如何解决，让模型调用MCP时取得更稳定的结果。在同时使用了多个MCP Server的时候，也可以用SKILL对多个编排多个MCP Server的工作流。

# 3.7 SKILL vs Prompts

Skills相比于Prompts的核心差别在于他们的**形态**和**生命周期**。
Prompts是一次性对话内容，你需要自己保存，复制和粘贴。模型不会知道什么时候用哪一个Prompt，而且你需要把Prompts **全部塞到上下文**里面。这会大量占用你的上下文空间。
Skills则是将提示词以一种更加工程化的包装方式打包，他是一个文件夹的形式，里面包含skills.md和相关的资源文件。他会动态加载/渐进式披露提示词，更加节省提示词。容易复用和版本管理，易于协作和共享。
总结来说，Skills不是一个更复杂的提示词，而是把复杂的提示词，变成了一个**有结构，可复用，可以自动调用的模块**。

| 侧重点 | 类比 | Token消耗 | 核心主体 | 编写难度 |
| :--- | :--- | :--- | :--- | :--- |
| **Agent Skills** | 提示词 | 带目录的说明书 | 低 | Markdown文件 | 低 |
| **MCP** | 工具调用 | 标准化工具箱 | 高 | 软件包 | 高 |

## 3.8 SKILL的运行模式

a. **用户发起任务**
用户对 Claude Code 提需求，Claude Code 把这个请求连同当前可用的 Skills列表结合元数据一起交给AI，让 AI先基于需求判断该用哪类能力。

b. **AI选择要用的 Skill**
AI 在可用的Skills里做匹配，然后返回指定的Skills

c. **按需加载 Skill 的指令层**
一旦选定 skill，Claude Code 才将该 skill 的具体定义交给AI，这一步体现了Skills指令层 **按需加载**：只有用到这个 skill 时，才把更详细的指令灌给模型。

d. **按需加载 Skill 的资源层 → 读范文/材料 → 生成最终回答**
如果需要使用到资源层，接下来Claude Code 会执行读资源的动作，把 references/范文内容喂给AI；AI 基于用户材料 + skill 指令 + references 完成写作并输出最终回答。

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   User                                          Claude Code      │
│                                                                 │
│   结合材料写作                                  User: 结合材料写作 │
│                                                 Available Skills: ………  │
│                                                                 │
│                                   Choose Skill: 帮我写作       │
│                                                                 │
│                                   Skill:                      │
│                                   - name: 帮我写作                │
│                                   - instruction:结合我发你的材料，撰写一篇深度文案， │
│                                   如果是软件/AI/计算机相关的材料，请先阅读参考       │
│                                   references/里面所有的范文，学习我的行文风格。 │
│                                                                 │
│   指令层 按需加载                                  │
│                                                                 │
│   资源层 按需加载                                  Read(.claude\skills\xxx\references\xxxx.md) │
│                                                                 │
│                                               范文内容         │
│                                                                 │
│                                               最终回答         │
│                                               给出了最终回答   │
│                                                                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## 3.9 SKILL的设计理念

（上面其实是一些SKILL的基础知识，相对来说是最基础，或者考到SKILL必问的。必须准备好，这一节可以理解为一个拔高项，我们不谈Skills的具体概念，而结合行业的痛点，发展趋势，未来的展望来聊Skills。如果你能和面试官讲出来，会体现你确实是有思考，有想法，关注着行业发展和趋势的。把这些答了，是加分项）

发展到现在（2026年初），现在的Agent工程的痛点不再能不能做，而是能不能交付。目前的行业痛点是：

1.虽然模型能力很强，但是常识很弱，能解决通用问题，却在具体业务问题上频繁踩雷。经常做的是文本补全，而不是业务理解。这通常导致造出来的Agent需要质检，返工，人力成本高。

2.现在经常导致的一个问题是，公司重复造轮子。给财务做一个Agent，法务做一个Agent，运营做一个Agent，每个Agent都有自己权限，工具，prompt等。就会变成同一个业务规则，在5个Agent里各写一遍。这就导致重复造轮子很严重，难以维护。

3.现在的Agent通常是一个会有复杂的prompts，这导致token太大，模型不能精准执行，另外成本上升。当你用复制粘贴扩展能力，系统就会用维护地狱来还债。

所以行业发展到现在，需要从堆砌提示词，迭代，调试，优化做一个Agent的过程变成做一个**可治理，可迭代，可复用的产品**。通过Skills，行业开发Agent的模式变成去写一个**可复用，可管理的技能资产（Reusable skills）**，结合通用内核，来解决这些痛点。

---

