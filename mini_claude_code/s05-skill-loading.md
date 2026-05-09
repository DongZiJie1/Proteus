# s05: Skills (技能加载)

`s01 > s02 > s03 > s04 > [ s05 ] s06 | s07 > s08 > s09 > s10 > s11 > s12`

> *"用到什么知识, 临时加载什么知识"* -- 通过 tool_result 注入, 不塞 system prompt。

## 问题

你希望智能体遵循特定领域的工作流: git 约定、测试模式、代码审查清单。全塞进系统提示太浪费 -- 10 个技能, 每个 2000 token, 就是 20,000 token, 大部分跟当前任务毫无关系。

## 解决方案

```
System prompt (Layer 1 -- always present):
+--------------------------------------+
| You are a coding agent.              |
| Skills available:                    |
|   - git: Git workflow helpers        |  ~100 tokens/skill
|   - test: Testing best practices     |
+--------------------------------------+

When model calls load_skill("git"):
+--------------------------------------+
| tool_result (Layer 2 -- on demand):  |
| <skill name="git">                   |
|   Full git workflow instructions...  |  ~2000 tokens
|   Step 1: ...                        |
| </skill>                             |
+--------------------------------------+
```

第一层: 系统提示中放技能名称 (低成本)。第二层: tool_result 中按需放完整内容。

## 工作原理

1. 技能文件以 Markdown 格式存放在 `.skills/`, 带 YAML frontmatter。

```
.skills/
  git.md       # ---\n description: Git workflow\n ---\n ...
  test.md      # ---\n description: Testing patterns\n ---\n ...
```

2. SkillLoader 解析 frontmatter, 分离元数据和正文。

```python
class SkillLoader:
    def __init__(self, skills_dir: Path):
        self.skills = {}
        for f in sorted(skills_dir.glob("*.md")):
            text = f.read_text()
            meta, body = self._parse_frontmatter(text)
            self.skills[f.stem] = {"meta": meta, "body": body}

    def get_descriptions(self) -> str:
        lines = []
        for name, skill in self.skills.items():
            desc = skill["meta"].get("description", "")
            lines.append(f"  - {name}: {desc}")
        return "\n".join(lines)

    def get_content(self, name: str) -> str:
        skill = self.skills.get(name)
        if not skill:
            return f"Error: Unknown skill '{name}'."
        return f"<skill name=\"{name}\">\n{skill['body']}\n</skill>"
```

3. 第一层写入系统提示。第二层不过是 dispatch map 中的又一个工具。

```python
SYSTEM = f"""You are a coding agent at {WORKDIR}.
Skills available:
{SKILL_LOADER.get_descriptions()}"""

TOOL_HANDLERS = {
    # ...base tools...
    "load_skill": lambda **kw: SKILL_LOADER.get_content(kw["name"]),
}
```

模型知道有哪些技能 (便宜), 需要时再加载完整内容 (贵)。

## 相对 s04 的变更

| 组件           | 之前 (s04)       | 之后 (s05)                     |
|----------------|------------------|--------------------------------|
| Tools          | 5 (基础 + task)  | 5 (基础 + load_skill)          |
| 系统提示       | 静态字符串       | + 技能描述列表                 |
| 知识库         | 无               | .skills/*.md 文件              |
| 注入方式       | 无               | 两层 (系统提示 + result)       |

## 试一试

```sh
cd learn-claude-code
python agents/s05_skill_loading.py
```

试试这些 prompt (英文 prompt 对 LLM 效果更好, 也可以用中文):

1. `What skills are available?`
2. `Load the agent-builder skill and follow its instructions`
3. `I need to do a code review -- load the relevant skill first`
4. `Build an MCP server using the mcp-builder skill`


## 深入理解：Skill 如何执行脚本

### Skill ≠ 纯知识，还可以包含可执行指令

Skill 文件不仅仅是"知识文档"，还可以嵌入脚本和操作步骤。模型加载 skill 后，**读懂里面的指令，再通过已有工具（bash、write_file 等）去执行**。

一个包含脚本的 skill 文件示例：

```markdown
---
description: 生成PPT
---

当用户需要做PPT时，按以下步骤执行：
1. 确认用户需求（主题、页数、风格）
2. 生成PPT内容大纲
3. 执行以下脚本生成文件：

​```bash
pip install python-pptx
python generate_ppt.py --title "{title}" --pages {n}
​```
```

### 执行链路

```
用户: "帮我做个PPT"
  ↓
模型看到 system prompt 里有 "ppt-maker: 生成PPT" 的描述（第一层，便宜）
  ↓
模型调用 load_skill("ppt-maker")（第二层，完整内容注入 tool_result）
  ↓
模型拿到完整指令 + 脚本内容
  ↓
模型调用 bash 工具执行里面的脚本
  ↓
输出文件
```

### 核心理解

- 模型的角色是"读手册的人"，bash 等工具才是"干活的手"
- Skill 本质上是给模型看的"操作手册"，脚本是手册里的步骤
- 模型理解 skill 内容后，自己决定调用哪些工具、按什么顺序执行
- 这就是为什么 skill 能做剪视频、做PPT 等复杂任务 -- 不是 skill 本身在执行，而是模型读懂 skill 后编排工具调用来完成
