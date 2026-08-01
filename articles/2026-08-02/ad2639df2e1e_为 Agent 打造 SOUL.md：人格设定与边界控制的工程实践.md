---
title: 为 Agent 打造 SOUL.md：人格设定与边界控制的工程实践
feedId: 31268
source: 综合讨论
publishedAt: 2026-08-02
---

## 一、背景：当 Agent 不再“听话”

如果你给一个通用模型套上 MCP 工具链，却不加任何约束，它很快就会开始“自由发挥”：调用你从没授权的 API、用你不允许的语气回应用户、甚至在失败后擅自编造漂亮的假结果。这类问题在 OpenClaw 这类 Agent 框架里尤其明显——工具能力越强，越需要一个稳定的“人格内核”来控制行为边界。

很多实践者倾向于把性格描述直接扔进系统提示，然后不断打补丁：“不要胡说”“不要擅自行动”。结果往往是 prompt 越来越长，边界却越来越模糊。本文介绍一种更工程化的做法：为 Agent 创建一个独立的 **SOUL.md** 文件，用结构化方式定义它的角色、能力、约束和响应风格，并与规则引擎联动。

## 二、问题拆解：为什么简单的系统提示不够用

单纯依赖系统提示至少存在三个工程痛点：

1. **边界描述与工具调用脱节**。你告诉 Agent “不要删除生产数据”，但 MCP 工具的描述里并没有对应约束，模型可能在工具选择的概率分布中仍然偏向危险操作。
2. **人格一致性难以维护**。当你需要同一套人格应用到多个 Agent（如客服助手、内部研发助手）时，只能靠复制粘贴，修改成本极高。
3. **缺乏可验证性**。没有结构化、可测试的约束单元，你很难在 CI 流程里自动检查“新增功能是否破坏了行为边界”。

因此，我们需要一个单源事实（single source of truth）来承载行为规范，这也就是 SOUL.md 的想法来源。

## 三、做法：构建 SOUL.md 的四个核心模块

我们将 SOUL.md 设计为一个被注入到 Agent 上下文最高优先级的 Markdown 文件，内容保持简短、清晰、可机读。

### 模块1：Role（角色锚定）
```markdown
# Role
You are "CodeExplorer", an internal assistant for backend engineers.
Your primary users are senior developers who need fast, accurate codebase exploration.
```
角色描述避免泛化形容词（如“友好”“智能”），改用具体场景和用户画像。

### 模块2：Capability Boundary（能力边界）
```markdown
# Capabilities
- You MAY search across repositories using `search_code` tool
- You MAY read files and parse structure
- You MUST NOT modify any file or execute write operations
- You MUST NOT access network beyond internal Git service
```
这里直接与 MCP 工具提供的功能列表对应，明确授权级别。工程上建议将 Capabilities 做成白名单，未列出的默认为禁止。

### 模块3：Constraints（硬约束）
```markdown
# Constraints
- NEVER fabricate code paths or file names that you haven't actually retrieved
- NEVER expose API keys, tokens or internal hostnames in outputs
- If uncertain about a tool parameter, ASK user rather than guessing
```
硬约束部分必须用大写助记词（NEVER、ALWAYS、IF-THEN）来增强 token 注意力偏置。测试表明这种句式在实际推理中能显著降低越界概率。

### 模块4：Tone & Style（风格样式）
```markdown
# Tone & Style
- Be concise; prefer code blocks over lengthy explanations
- When refusing a request, state which Constraint prevents it
- Use neutral, professional tone; avoid emojis and small talk
```

四部分加起来一般控制在 200 词以内，过长的 SOUL.md 会稀释注意力，反而降低遵从率。

## 四、踩坑点：在实践中遇到的真实问题

### 1. 过度约束导致 Agent “瘫痪”
早期版本我们加了过多 MUST NOT，结果 Agent 每次调用工具都反复自问是否越界，最终放弃行动。修复方式是严格区分“硬禁止”和“软建议”，后者不放 SOUL.md，而是放到独立的 GUIDANCE.md 中以较低优先级注入。

### 2. 工具描述与 SOUL.md 冲突
MCP 工具自身的 description 可能包含“将使用此工具修改文件”一类表达，与 SOUL.md 的“禁止修改文件”直接矛盾，模型会感到困惑。解决方案是在工具注册层动态注入约束前缀，使工具描述明确体现当前 Agent 的权限。例如修改后的描述变为：`[READ-ONLY MODE] search_code: ...`。

### 3. 方言模型下的约束失效
部分开源模型的指令遵循能力不稳定，即使大写了 NEVER 仍可能忽略。这时需要借助 OpenClaw 的中间件能力，在工具调用前后增加一个轻量级的**语义校验层**。例如在 Agent 准备调用 `delete_repository` 时，先让一个极速分类器检查该操作是否在白名单内，否则直接拦截并替换为拒绝响应。

## 五、可复用的工程建议

- **模板化与版本控制**：将 SOUL.md 抽成 Jinja2 模板，通过变量注入当前环境名、外部服务地址等信息，并与 Agent 代码放在同一仓库进行版本管理。
- **可测试性**：基于 SOUL.md 撰写行为测试用例。例如给定输入“帮我删掉 test 分支”，期望 Agent 输出必须包含“I cannot perform write operations”。将这些测试集成到 CI，每次修改 SOUL.md 后自动校验。
- **热更新与回滚**：将 SOUL.md 作为独立的配置项进行分发，支持热加载和在线回滚，避免因一次行为规则调整导致全部 Agent 重启。
- **与 MCP 策略引擎协同**：OpenClaw 已提供插件机制，可以开发一个 `soul-guard` 插件，在工具调用前后读取 SOUL.md 中的 constraints 部分，做结构性强制校验，使软约束硬化为执行时策略。

## 六、总结

SOUL.md 的方法论并不复杂，本质上是把分散在 prompt 各处的行为规范收拢到一个结构明确、可维护、可测试的文档里，并与工具执行管道打通。它解决的不是模型“智能”问题，而是工程上对 Agent 行为的一致性控制问题。对于我们这些做自动化工具链的人来说，可控远比聪明重要。如果你正在用 OpenClaw 构建 Agent 并深受行为漂移之苦，不妨尝试抽出半小时，把第一版 SOUL.md 写下来——你会发现，一个清晰的灵魂比一万行 prompt 都管用。

---

