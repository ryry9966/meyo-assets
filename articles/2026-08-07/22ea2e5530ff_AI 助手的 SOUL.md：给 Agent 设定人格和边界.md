---
title: AI 助手的 SOUL.md：给 Agent 设定人格和边界
feedId: 31904
source: 综合讨论
publishedAt: 2026-08-07
---

# AI 助手的 SOUL.md：给 Agent 设定人格和边界

在自动化实践里，Agent 的行为一致性始终是个头疼的问题。system prompt 散落在代码字符串、配置文件或团队文档里，多人协作时极易出现版本漂移。尤其当 Agent 挂载了 MCP 工具后，能力边界变得模糊，一句模糊的自然语言约束很难拦住模型对敏感工具的调用。经过几次生产事故后，我们开始用一种简单但有效的模式来收敛问题：**把 Agent 的人格、沟通风格、能力范围与硬边界统一存放在一个 `SOUL.md` 文件中，并将其作为系统消息的核心组件加载。**

这篇文章面向正在使用 OpenClaw、MCP 或相关插件体系构建 Agent 的同行，分享这种做法的工程细节与踩坑经验。

## 一、背景与问题

典型的 Agent prompt 构造方式是字符串拼接：把角色描述、能力列表、注意事项塞进一段文本，直接作为 messages 的第一条。当 Agent 只做一两个固定任务时，这没什么问题。一旦业务复杂化，问题就接连出现：

- **散布在多处**：工具调用规则写在代码里，语气控制写在前端模板，安全约束又放在运维文档。修改一处很难同步到所有相关位置。
- **版本不可控**：prompt 调整往往直接发版，缺乏评审和回滚机制，很难追溯一个异常回复到底是由哪版 prompt 引起的。
- **边界弱约束**：对「不要执行危险命令」这类限制，自然语言很容易被对抗性提示绕过，尤其当 Agent 具备 Shell 或文件系统访问能力时。
- **测试困难**：没有单一事实来源，自动化测试难以断言「Agent 在收到某类输入时是否严格遵守了既定人格和边界」。

这些问题的根因是缺少一个**可版本化、可评审、可测试的人格定义层**。

## 二、SOUL.md 的结构设计

`SOUL.md` 本质上是一份结构化的 Markdown 文件，强制将 Agent 的行为契约拆成几个固定的段落。我们内部沉淀出一套最小模板：

```
# Identity
- 名称/代号
- 服务对象（内部团队/客户）
- 核心定位（故障排障助手/代码审查/数据查询）

# Communication Style
- 语气（正式/随意）
- 输出格式偏好（纯文本/Markdown/JSON）
- 禁止使用的词汇或表达方式

# Capabilities
- 明确允许的能力清单（例如：查询日志、重启服务）
- 每项能力关联的具体工具或 MCP 服务名称

# Constraints (硬边界)
- 绝对禁止的操作（如执行 rm、修改生产数据库）
- 信息脱敏规则（不返回邮箱、手机号）
- 上下文使用限制（不能记忆用户数据跨会话）

# Examples (optional)
- 若干对话示例，展示在灰色地带 Agent 应如何响应
```

这份文件放在 Agent 项目的根目录下，纳入版本控制。结构化不是为程序解析，而是**约束人写的范围**，让团队成员知道该往哪里填、该补什么规则。

## 三、在 OpenClaw 中加载 SOUL.md

OpenClaw 的 Agent 配置可以在 `agent.config.yaml` 中指定外部 prompt 文件。我们利用这一特性，将 `SOUL.md` 作为系统消息主体挂载：

```yaml
# agent.config.yaml
agent:
  name: ops-buddy
  system_message: file://./SOUL.md
  tools:
    - mcp: log-fetcher
    - mcp: service-controller
  tool_permission:
    service-controller.restart:
      requires_confirmation: true
```

框架启动时会读取 `SOUL.md` 内容，注入到系统消息最前端。对于需要动态变量的场景，使用简单的模板渲染（如 Mustache）在加载时替换 `{{user_context}}` 等运行时信息。

如果是自己拼接消息的逻辑，做法更直接：

```python
soul = Path("SOUL.md").read_text(encoding="utf-8")
messages = [
    {"role": "system", "content": soul},
    {"role": "user", "content": user_input}
]
```

无论哪种方式，一个核心原则是：**SOUL.md 必须是唯一的系统级 Prompt 来源，其他部分只能通过工具描述或动态注入临时规则，不得重复定义人格和边界。**

## 四、与 MCP 工具边界的协同

MCP 让 Agent 能够调用外部服务，但工具侧的权限控制在 Server 端通常是粗粒度的（允许/禁止使用某工具）。真正危险的往往是「允许使用，但需约束参数或使用场景」。这时 `SOUL.md` 的 Constraints 段就起到了第二层防线的作用。

比如我们在 SOUL.md 中明确规定：
> 当用户请求重启 production 标签的服务时，必须先输出一条 warning 并等待用户回复 confirm，不得直接调用 service-controller.restart。

这属于行为规范，无法在 MCP Server 的 JSON Schema 里表达，只能靠 Prompt 实现。但要注意，**Agent 的边界防线必须由「Prompt 约束」和「工具侧硬控制」共同构成**。SOUL.md 解决的是前者，后者仍需通过 MCP Server 的权限配置或 OpenClaw 的 `tool_permission` 来兜底。

## 五、踩坑记录

1. **文件过长导致 Token 浪费**  
   最初我们把所有对话示例都塞进 SOUL.md，典型场景下文件超过 2000 Token。每次对话都会占用大量上下文空间，挤压了实际任务的处理能力。后来把示例精简到 2-3 轮，其余放到外部文档链接里，只在训练或测试时使用。

2. **自然语言边界被绕过**  
   曾有一版 SOUL.md 写了「不要泄露用户邮箱」，但在连续推理中，Agent 先输出姓名，再被追问时补上了邮箱。对抗测试后，我们在 Constraints 里补充了脱敏检查规则，同时在 MCP 工具返回的数据层做了字段过滤。

3. **多人维护时的格式退化**  
   随着规则变多，有人开始在 Constraints 里写散文，导致后续难以快速查阅。我们后来强制要求 Constraints 使用无序列表，每一条一个 prohibition，并用 CI 脚本检查格式。一旦发现非列表行就提醒修正。

4. **热更新不生效**  
   刚开始在容器里改了 SOUL.md 直接重启 Agent 进程，没注意配置缓存。导致一段危险调用规则在线上延迟生效。现在在 OpenClaw 配置中开启了文件监听模式，变更后自动重载系统消息，同时记录 prompt 版本号到日志。

## 六、可复用建议

- **模板优先**：为团队提供带注释的 SOUL.md 模板，降低新人填写成本。
- **动静分离**：运行时变量（如日期、用户角色）通过占位符注入，不直接写在 SOUL.md 里。
- **与 CI 联动**：每次 PR 改动 SOUL.md 时，自动运行对抗测试集，验证 Agent 是否仍遵守核心边界。
- **硬边界加注释**：在文件内用 `<!-- HARD CONSTRAINT -->` 注释标记完全不能妥协的规则，方便未来自动化解析和监控。
- **多 Agent 场景用子目录**：如果同一仓库维护多个 Agent，使用 `agents/ops-buddy/SOUL.md` 结构，避免混淆。

## 七、总结

`SOUL.md` 本质上是一次对 Agent 行为约定的「工程化收拢」。它不解决所有安全问题，也不能替代权限控制和输出验证，但它给团队提供了一个明确、可讨论、可演进的基座。当你的 Agent 开始从简单脚本变成需要多人协作维护的服务时，花半小时抽出第一版 SOUL.md，会省下后面成倍的排障和扯皮时间。

---

