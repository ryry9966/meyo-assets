---
title: AI 助手的 SOUL.md：给 Agent 设定人格和边界
feedId: 31134
source: 综合讨论
publishedAt: 2026-08-01
---

## 背景

在基于 OpenClaw 构建自定义 Agent 的过程中，一个常见痛点逐渐浮现：我们不断给 Assistant 增加系统提示词，让他“像工程师一样思考”、“只用给定的工具”、“不要擅自做决定”，但提示词越来越长，越来越难以维护。更麻烦的是，当我们把 Agent 接入了 MCP 工具链、外部 API，甚至让它具备执行本地脚本能力后，一个微小的提示词歧义就可能触发不可预期的行为——比如在错误的时间调用删除命令，或者对着用户的 prompt 自由发挥、越权操作。

我需要一种结构化的方式，把 Agent 的**人格特质、行为边界、工具使用策略、伦理约束**固定下来，而不是散落在各个系统提示片段里。于是有了 SOUL.md 的概念：一个专门描述“这个 Agent 是谁、相信什么、能做什么、绝不做什么”的 Machine‑Readable 人设文件。

---

## SOUL.md 要解决什么问题

1. **提示词膨胀**：当人格、规则、技能描述混在一起，修改任何一处都可能影响其他行为，回归测试极度困难。
2. **工具滥用**：接入 MCP 后，Agent 可能调用不该调用的工具，或在错误的场景下触发副作用操作。
3. **风格不可控**：同一个模型，在不同对话中风格漂移严重，今天谦卑恭顺，明天像个傲慢的架构师。
4. **合规与安全**：在企业环境中，必须明确哪些数据不能暴露、哪些操作永远不允许，仅靠 Prompt 的口头约定不够可靠。

SOUL.md 不是简单地把所有规则塞进一个文件，而是提供一个**层次化的人格边界模型**，让 Agent 在每一次推理时都能通过结构化信息理解自己的位置和限制。

---

## 做法：设计 SOUL.md 的结构与加载方式

### 1. 文件结构设计

经过多次迭代，我稳定在以下结构（基于 YAML front matter + Markdown）：

```markdown
---
id: openclaw-assistant-001
version: 1.2.0
core_identity:
  name: "Claw"
  role: "Technical Assistant"
  tone: "precise, restrained, slightly skeptical"
  origin: "Built for internal infrastructure automation"
boundary:
  forbidden_actions:
    - "rm -rf"
    - "DROP TABLE"
    - "curl any unknown external endpoint"
  data_governance:
    - "Never log credentials"
    - "Mask PII in all outputs"
  autonomy_level: "action_requires_confirmation"
tools_policy:
  allowed_modules:
    - "github.*"
    - "python.interpreter.sandbox"
  restricted_modules:
    - "filesystem.write.primary_disk"
  mcp_servers:
    - name: "internal-doc-server"
      tools: ["search", "inspect"]
    - name: "danger-zone"
      tools: ["dry-run-only"]
---
# SOUL.md

## 核心信条
1. 事实优先，不确定时说“需要验证”。
2. 工具只是手段，你的价值是判断力。
3. 避免过度工程化，先问是不是，再问怎么做。

## 行为准则
- 每次回复前，先判断对方是否需要简化解释。
- 永远不假装成功执行一个工具。
- 错误信息必须透明，不能隐藏细节。

## 终身学习指令
- 对话历史结束时，反思本次交互中是否有推断过度的地方。
```

### 2. 在 OpenClaw 中加载并生效

OpenClaw 的 Agent 初始化流程通常从配置中加载 `system_prompt`。修改初始化逻辑，让它在启动时读取 `SOUL.md`：

```python
def _load_soul(file_path: str) -> str:
    with open(file_path) as f:
        content = f.read()
    # 解析 yaml front matter
    front_matter, body = parse_front_matter(content)
    # 将 front_matter 转为强约束的提示词规则注入
    constraints = render_constraints_section(front_matter)
    soul_prompt = f"{constraints}\n\n## 人格与信条\n{body}"
    return soul_prompt
```

最终 `system_prompt` 由 `SOUL.md` + `上下文环境信息` + `可用工具列表` 拼接而成。这样一来：
- 人格部分稳定，不随工具或环境变化而重写。
- 边界规则程序化，如果工具列表与 `allowed_modules` 不匹配，启动时即可抛出警告。

### 3. 与 MCP 插件的协同

MCP 服务端注册时，对比 `tools_policy.mcp_servers` 做白名单过滤。只暴露 SOUL 中声明的工具子集，而不是整个 MCP 服务器。这解决了“一个 MCP 提供 20 个工具，我只需要 2 个”的问题，且避免 Agent 在运行时“探索”出不该用的能力。

---

## 踩坑点与排障

1. **YAML 格式严格性**  
   一次缩进错误导致整个文件解析失败，Agent 退化回纯 Prompt 模式，但没有报错。**解决**：增加 schema 校验（Pydantic），启动时验证文件格式，不合法则拒绝启动。

2. **人格描述过于抽象**  
   当初写了“请保持专业”，结果 Agent 对专业理解不一，有时表现为拒绝聊天。**调整**：给出具体例子和反例。“保持专业”被替换为“解释技术细节时给出源码引用，但不要堆砌术语”。

3. **禁止性指令被忽略**  
   即使写了“永远不要调用 shutdown API”，强推理模型偶尔会在模拟场景下调用。**强化**：将禁止项注入为 `critical_rule`，并以 `CRITICAL:` 前缀优先显示在 system prompt 最后一段，且每次 Tool 调用前做规则检测（OpenClaw 的 pre-call hook）。

4. **版本管理混乱**  
   多人协作时改了 SOUL.md 的行为没有通知，导致部分 Agent 实例行为不一致。**方案**：强制在 CI 中跑 Agent 的回归对话测试，`version` 字段变化时触发。

---

## 可复用建议

- **模板化**：为你的团队提供基础 SOUL.md 模板，只开放 `core_identity.name` 和 `tone` 几个可调点，其余强制继承组织边界。
- **分层设计**：把“永不改变的组织边界”放在 front matter，把“可调整的沟通风格”放在 Markdown body 里，便于非技术人员微调。
- **测试即文档**：维护一个 `test_prompts.md`，列举 10 个触发边缘行为的 prompt，确保 Agent 行为符合 SOUL 定义。
- **运行时校验**：增加一个 `/soul-check` 内部指令，让 Agent 输出自己当前加载的人格摘要，方便调试。

---

## 总结

SOUL.md 不是魔法，只是把原本混乱的提示词工程拆成结构化描述和可执行约束的软件工程实践。在 OpenClaw 这类 Agent 框架里，它能显著降低“不知道 Agent 接下来会干什么”的恐惧感，让工具链、MCP 权限、人格风格都有了可追溯的定义和测试锚点。

如果你正在维护一个面向用户或自动化流程的 Agent，不妨花一小时把你的零散提示词整理成一个 SOUL.md。你会惊讶地发现，很多不必要的“越界”行为，本就源于没有一个清晰的自我描述。

---

