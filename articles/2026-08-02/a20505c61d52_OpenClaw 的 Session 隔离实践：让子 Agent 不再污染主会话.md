---
title: OpenClaw 的 Session 隔离实践：让子 Agent 不再污染主会话
feedId: 31367
source: 综合讨论
publishedAt: 2026-08-02
---

## 背景

在 OpenClaw 的多 Agent 协作场景中，我们经常通过插件或 MCP 服务将一部分任务委派给“子 Agent”处理。这些子 Agent 可能是本地脚本、专用模型或外部 API，被抽象为工具或独立的 Agent 实例。问题在于：默认的会话管理中，子 Agent 的调用记录（包括返回的完整内容）往往会直接进入主 Agent 的记忆链，导致主会话上下文被大量中间信息污染。长期运行的 Agent 系统尤其会因此产生注意力偏移、关键指令丢失，甚至让模型在后续推理中“钻进”子 Agent 的返回数据里，做出错误决策。

## 问题拆解

以一个实际场景为例：主 Agent 负责用户意图理解与对话管理，同时通过一个子 Agent（文档检索工具）查询长文本。如果子 Agent 每次返回数千字的原始片段，且这些片段都原样追加到主会话的 `messages` 列表中，那么：

- 主 Agent 在下一轮对话中会将大量无关细节纳入权重计算；
- Token 消耗急剧上升，成本不可控；
- 当检索结果质量波动时，主 Agent 可能输出幻觉或遗忘用户初始要求；
- 难以在多轮交互中维持稳定的行为模式。

根本矛盾在于：**子 Agent 的输出是为了完成局部任务，不应直接成为主会话的长久记忆。** 我们需要在“传递必要信息”和“避免上下文污染”之间找到边界。

## 做法与步骤

OpenClaw 提供了灵活的会话管理和 Agent 协作机制，可以通过组合配置和少量代码实现子 Agent 的 session 隔离。以下是我在生产环境验证过的可行路径。

### 1. 识别需要隔离的节点

不是所有子 Agent 都必须隔离。评估标准：

- 返回内容长度 > 200 tokens 且为辅助性数据；
- 子 Agent 任务与主对话主线关联弱，只在当前步骤有意义；
- 子 Agent 运行后，主 Agent 只需要结构化摘要或执行反馈，不需要原始输出。

对于这类子 Agent，启用隔离收益明显。

### 2. 配置独立 Session 并控制消息注入

OpenClaw 的 Agent 配置支持 `session` 和 `memory` 策略。我推荐为子 Agent 声明 `inherit_memory: false` 并设置 `context_policy: isolated`。示例配置片段：

```yaml
agents:
  assistant:
    model: openai/gpt-4.1
    tools: [search_agent]
  search_agent:
    model: openai/gpt-4.1-mini
    inherit_memory: false
    context_policy: isolated
    max_tokens: 800
```

当 `assistant` 调用 `search_agent` 时，子 Agent 只会看到本次调用传入的指令，无法读取主会话的完整历史。这能防止它返回不必要的上下文关联信息。同时，`context_policy: isolated` 会让 OpenClaw 在子 Agent 执行完毕后自动裁剪其返回内容，只保留必要摘要（你可以通过 `retention` 规则进一步定制）。

### 3. 在调用层做消息裁剪（Middleware）

如果子 Agent 是通过插件方式引入，或者你需要更细粒度的控制，可以借助 OpenClaw 的中间件机制。在 `before_tool_call` 阶段，将传入的消息列表压缩为最小必要提示；在 `after_tool_call` 阶段，对工具返回结果进行二次加工，只提取结构化数据。

摘录一段中间件示例（TypeScript 伪代码）：

```typescript
openclaw.on('tool:beforeCall', async (ctx) => {
  if (ctx.toolName === 'deepSearch') {
    // 只保留最近一轮用户消息和当前工具调用意图
    ctx.messages = ctx.messages.slice(-2);
  }
});

openclaw.on('tool:afterCall', async (ctx) => {
  if (ctx.toolName === 'deepSearch' && ctx.result) {
    // 将长文本替换为压缩摘要，避免污染主会话
    ctx.result = { summary: summarize(ctx.result.text) };
  }
});
```

这样做的好处是：主 Agent 看到的是 `summary` 而非原始文本，主记忆链依然干净。

### 4. 利用 MCP 的响应契约限制返回长度

当子 Agent 以 MCP 工具形式接入时，可以在工具描述中通过 `response_schema` 强制限定返回的 JSON 结构，并在服务端截断长内容。例如，在 MCP 服务定义中设置 `maxResponseTokens` 或返回时主动 trim。确保主 Agent 拿到的始终是可控的结果，避免通篇 chunk 进入 session。

## 踩坑点

1. **隔离过狠导致子 Agent 缺少必要上下文**  
   有个场景是子 Agent 需要知道当前用户的角色权限才能正确过滤结果。完全隔离后它拿不到用户角色信息，返回了无权限开放的数据。解决方案是在调用参数中显式传入所需上下文，或在调用前的裁剪逻辑中加入对应的字段。

2. **子 Agent 返回错误后主 Agent 行为紊乱**  
   如果子 Agent 返回了异常，且异常信息直接被喂给主模型，很容易诱导模型编造解释。建议在 `afterCall` 中统一捕获错误，并以固定格式返回（例如 `{error: true, hint: "暂时无法获取数据"}`），避免错误堆栈进入记忆。

3. **独立 Session 的生命周期管理**  
   启用隔离后，子 Agent 的 session 如果不主动清理，可能大量堆积占用存储。OpenClaw 支持 `session_ttl`，建议为其设置较短的过期时间（如 5 分钟），任务完成即释放。

## 可复用建议

- **最小必要信息原则**：只向子 Agent 传递完成当前任务所必需的数据，包括用户意图，但不包括历史闲聊。
- **结构化返回**：强制子 Agent 以 JSON 或短摘要形式返回，而非自然语言长篇大论。这也有利于主 Agent 做条件判断。
- **接入层统一处理**：编写可复用的 `isolationMiddleware`，通过配置开关决定哪些工具需要隔离，避免到处散落硬编码。
- **监控与日志**：记录子 Agent 注入主会话的 token 数量，超额时告警，便于持续调优。

## 总结

OpenClaw 的 session 隔离不是一个“开启开关”就搞定的事情，而是需要配合上下文策略、中间件裁剪和返回约束的工程化组合。按照上述实践，我成功将生产环境中主 Agent 的单次调用 token 消耗降低了 40% 以上，且幻觉率显著下降。**让子 Agent 专心干活，让主 Agent 保持清醒**，这才是稳态多 Agent 系统的基础。

---

