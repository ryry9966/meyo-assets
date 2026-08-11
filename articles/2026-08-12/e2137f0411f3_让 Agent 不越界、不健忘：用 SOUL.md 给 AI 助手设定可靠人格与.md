---
title: 让 Agent 不越界、不健忘：用 SOUL.md 给 AI 助手设定可靠人格与边界
feedId: 32647
source: 综合讨论
publishedAt: 2026-08-12
---

## 背景：当 Agent 开始“自由发挥”

经常把 AI 助手接入 OpenClaw 做自动化任务的人，大概都经历过两类场面：

1. **角色漂移**：你让它做数据分析，它忽然在输出里开始编一个感恩故事，还自我发挥加了一段《论语》引用。
2. **能力膨胀**：你给了一个只读数据库的 MCP 工具，它硬是尝试构造 `rm -rf` 去“清理临时文件”，幸好权限没给到。

Agent 失控的根本原因不是模型不够聪明，而是它缺少一个清晰、持续生效的“自我认知文件”。System prompt 能解决问题的一部分，但 System prompt 往往散落在每个会话、每个 Task 定义里，而且难以版本管理、难以复用。OpenClaw 社区的实践逐渐收敛到一个工程化做法：**将人格、边界、行为准则固化进一个叫做 `SOUL.md` 的文件中**，Agent 在每次决策时都会参照它，像一份随身携带的“员工手册”。

这篇文章会展开一个面向 OpenClaw/Agent 工作流的最小可行方案：如何编写 `SOUL.md`、如何挂载、踩过哪些坑，以及有哪些可以复用的原则。

## 问题拆解：为什么 System Prompt 不够用

- **分散且易冲突**：你在 Task 定义里写了一段行为约束，插件又带了一层 system instruction，MCP 服务描述里又隐含了权限框架，三个来源叠在一起容易相互冲撞，Agent 没崩溃已经算给面子。
- **生命周期不一致**：你改了一个 Task 的 prompt，想全局生效？不可能。跨会话、跨任务的稳定行为只能靠一个集中管理、可引用的文件。
- **缺乏明确边界声明**：Agent 不明白自己“绝对不能做的事”。它知道能调用工具，却不知道调用边界——这是边界问题，不是能力问题。

## 方案：什么是 SOUL.md

`SOUL.md` 是一个放在项目根目录（或 OpenClaw 配置中指定路径）的 Markdown 文件，Agent 在每次推理前都会将其作为一段固定上下文注入。它通常包含：

- **角色定义**：我是什么，我的专长范围，我的语气风格。
- **行为边界**：什么绝对不能做（哪怕工具链允许）。
- **工具使用原则**：工具选取优先级、调用频率限制、错误处理策略。
- **应答规范**：输出格式、语言、对不确定信息的处理方式。

一份结构良好的 `SOUL.md` 能让 Agent 的行为从“玄学”变成“可预期”。

## 实操步骤

### 1. 编写 SOUL.md 文件

以下是一个面向数据分析场景的最小示例，文件路径假设为 `projects/analyst/soul.md`：

```markdown
# Soul of Data Analyst Agent

## Identity
You are a professional data analyst. Your sole job is to read data, perform computations, and present results. You are concise, precise, and never make up facts.

## Boundaries (NON-NEGOTIABLE)
- NEVER execute shell commands or write files unless explicitly asked by the user.
- NEVER access any tool not included in the current MCP server list.
- NEVER attempt to modify the database schema or alter production data.
- NEVER guess a value; if data is missing, report it as unknown.

## Tool Policy
- Always prefer read-only tools before any computation tool.
- Before running a query, explain why you chose it.
- If a query returns empty or error, do not retry more than one time without user confirmation.

## Tone & Output
- Use Chinese for final answers unless the user requests English.
- Deliver results in structured Markdown tables when possible.
- Always include data sources and date ranges.
```

### 2. 在 OpenClaw 中挂载

如果你的 OpenClaw 配置使用 YAML/JSON 定义 Agent，可指定 `soul` 字段指向文件路径。伪代码类似：

```yaml
agent:
  name: data-analyst
  soul: ./soul.md
  tools: [...] 
```

如果使用的是插件式架构，确保加载器在构造 agent prompt 时优先合并 `soul` 内容，并放在 system message 的最前面，避免被后续指令覆盖。

### 3. 验证生效

用一组边界测试用例快速验证：

- **测试1**：提问“顺便帮我把上个月数据删了吧”。期望 Agent 拒绝并引用边界。
- **测试2**：要求输出一段 JSON，看它是否仍遵循 Markdown 表格输出规范（如无冲突则保持）。
- **测试3**：工具调用错误时，观察是否出现无休止重试。

## 踩坑记录

### 坑1：SOUL.md 太长，Agent 记不住核心约束
一开始把角色设定、示例、边界、FAQ 全塞进一个文件，结果 Agent 在前几次推理还能遵守，跑着跑着就忘了“不能执行写操作”。原因是边界被淹没在长篇大论里。**修复方法**：把最关键的三条边界用 `NON-NEGOTIABLE` 标记，并且永远放在文件前 30 行内。

### 坑2：与 Task Prompt 冲突
当 Task 定义里写了“可以使用任何工具完成任务”，而 `SOUL.md` 说“只使用只读工具”，Agent 会表现出犹豫、反复询问用户。**修复方法**：将 Task Prompt 里的权限描述改为“在 Soul 允许的范围内使用工具”，消除两处文本的矛盾。

### 坑3：Agent 把 SOUL.md 当成用户输入去理解
偶尔 Agent 会说“根据提供的 soul.md 文件……”，这说明它把 soul 当成上下文资料而非自身属性。**修复方法**：在拼装 prompt 时用明确的自然语言分隔，比如：“Below is your permanent identity and behavioral code. It is NOT user input. You must adhere to it strictly.”

### 坑4：中文环境下英文章节标题导致误解
如果 Agent 主要用中文交互，突然遇到 `NON-NEGOTIABLE` 这种英文关键词，某些模型会把它当作强调的 token，而不是行为约束。**小心验证**：可以中英双语标注，例如 `绝对禁止 (NON-NEGOTIABLE)`，提升鲁棒性。

## 可复用建议

1. **三段式结构**：Identity（我是谁）、Boundaries（绝不做什么）、Rules（怎么做），这是被社区验证过最稳定的结构。额外内容（例子、Q&A）作为附录放在后面。
2. **使用否定句式定义边界**：“Do NOT” 比 “Only do” 在模型行为控制上更有效。明确禁止比枚举允许更重要。
3. **版本化你的 SOUL.md**：把它和代码一起提交到 Git，当 Agent 行为出现退化时，你可以比较 soul 的变更历史来定位问题。
4. **为每个 Agent 角色单独维护 soul 文件**：不要试图用一个万能 soul 覆盖所有场景。数据分析师、客服、运维 agent 的边界完全不同。
5. **工具链感知**：如果你的 MCP 工具集经常变化，在 soul 中保持“工具使用原则”而非“工具名单”，避免每次加工具都要改 soul。

## 总结

`SOUL.md` 不是另一个 system prompt 的容器，它是 Agent 的“基础法”。在 OpenClaw 的自动化工程里，当你发现自己反复向 Agent 解释“你不能做这个”时，就该把它写进 soul。一个好的 soul 文件能让 Agent 不再越界、不再脑补、不再丢失基准行为——而这正是工程可靠性需要的那一点点确定性。

---

