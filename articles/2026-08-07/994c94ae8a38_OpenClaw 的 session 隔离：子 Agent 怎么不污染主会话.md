---
title: OpenClaw 的 session 隔离：子 Agent 怎么不污染主会话
feedId: 31962
source: 综合讨论
publishedAt: 2026-08-07
---

## 背景：当子 Agent 开始“接话”

在 OpenClaw 里，我们经常会通过插件或 MCP 工具把任务委派给一个子 Agent。子 Agent 的分析过程、工具调用、中间步骤，默认情况下会直接注入当前 conversation context。这在简单场景里还好，一旦子 Agent 的输出又长又杂，主会话很快就被冲得七零八落，后续的 prompt 被大量无关信息占据，token 消耗飙升，主 Agent 的判断也会被带偏。

更致命的是，如果子 Agent 的推理过程里出现了某些“先入为主”的判断，或者它以“我”的口吻在对话里插入了新的诉求，主 Agent 很可能就顺着它走，把用户意图完全带偏。

**核心诉求很明确**：子 Agent 完成它的活儿，只把最终要的结论、结构化数据或异常回调交给主会话，中间全过程的“噪声”一概隔离。

## 问题拆解：Session 混入是怎么发生的

OpenClaw 默认情况下，所有 plugin 或 tool 返回的内容都是直接 append 到当前 `messages` 数组的。当你调用一个子 Agent（例如用 `request_analysis` 工具）时，如果你的工具实现直接返回了子 Agent 的整段对话历史，OpenClaw 就会把它当作用户/助手消息的一部分继续往下传。

这会导致三个问题：

1. **Context 膨胀**：子 Agent 的思考链被原样灌进主会话，token 成本高，openai/claude api 调用容易撞上下文窗口。
2. **角色混淆**：子 Agent 以 assistant 口吻输出的话，主 Agent 可能误以为是自己的上一轮发言。
3. **工具路由紊乱**：子 Agent 发起的 tool call 可能和主会话里的工具调用 schema 混在一起，导致解析错误。

## 做法：给子 Agent 一个独立的 Session，只抽取结论

**思路**：为每个子 Agent 调用创建一个隔离的子会话（sub-session），子 Agent 在里面对话、用工具，完成后只把最终用户需要的结果、结构化摘要或错误信息注入主会话。

### 步骤 1：创建子 Session

在你的工具实现里，不要复用当前 `ctx` 的 conversation，而是显式创建一个新的 session。OpenClaw 内部提供了 `createSubSession` 或相关能力，可以从当前上下文中派生一个干净的对话环境。

伪代码示例（Node.js 插件）：

```javascript
const subSession = await ctx.createSubSession({
  systemPrompt: "你是负责分析用户数据的子Agent，只输出结构化JSON，不要闲聊。",
  tools: ["search_database", "summarize"],
  model: "gpt-4o-mini", // 子 Agent 用轻量模型
});
```

### 步骤 2：向子 Session 注入明确的任务 Prompt

把用户的主指令翻译成子 Agent 能理解的子任务。尽量不要把主会话的整段对话原封不动扔给子 Agent，而是提取关键意图。

```javascript
const prompt = `
请根据以下用户需求分析数据：
用户问题：${userQuery}
请返回：
{
  "finding": "核心发现",
  "confidence": 0.85,
  "suggestions": []
}
仅输出JSON，不要附加说明。
`;
const response = await subSession.run(prompt);
```

这一步里 `subSession.run()` 是阻塞的，但内部会进行多轮工具调用，最终返回一个完整的 conversation 对象，包含子 Agent 和 tool 之间的所有交互。

### 步骤 3：从子 Session 中抽取“交付物”

子 session 的完整 history 我们不会直接交给主会话。只提取最后 assistant 的最终回复，或者进行一步后处理（例如解析 JSON）。

```javascript
const finalMessage = response.messages[response.messages.length - 1].content;
let parsed;
try {
  parsed = JSON.parse(finalMessage);
} catch {
  // 如果解析失败，只回传一个简化的错误摘要
  parsed = { error: "子Agent返回格式异常", raw: finalMessage.slice(0, 200) };
}
return parsed;
```

主会话中你的工具只返回这个结构化的 `parsed` 对象，主 Agent 只看到干净的结论。

### 步骤 4：处理异常与超时

子会话可能出现死循环、工具调用失败、超时。必须加上防护：

```javascript
const timeoutMs = 20_000;
const result = await Promise.race([
  subSession.run(prompt),
  new Promise((_, reject) => setTimeout(() => reject(new Error('subSession timeout')), timeoutMs))
]);
```

超时或异常时，返回一个标准化的错误对象，让主 Agent 能感知到“子任务失败了”，视情况决定是要重试、降级还是向用户报告。

## 踩坑点

**1. 不要在子 Session 里直接用主 Session 的 tools**

主会话的工具可能依赖一些只在主会话上下文里存在的变量（例如 conversationId、userId）。如果给子 session 挂同样的工具，必须确保这些依赖是显式传参的，否则会出现静默失败。

建议给子 Session 专用的工具集合，或者给工具加一层 wrapper 把上下文变量序列化进去。

**2. 子 Session 的系统 prompt 要严格控制输出格式**

只需要一次“子 Agent 自由发挥”，你就要频繁写后处理逻辑去抠结构化数据。建议 prompt 里反复强调“只输出 JSON，不要任何解释”，并给出 JSON Schema 示例。可以用 `response_format: { type: "json_object" }` 强制模型遵守。

**3. Session 清理与资源释放**

子 Session 的上下文如果不手动释放，会一直占用内存。OpenClaw 里根据实现不同，有的在 run 结束后自动清理，有的需要调用 `ctx.destroySubSession(subSessionId)`。务必查看对应版本的 API 文档。

**4. 主会话 Context 仍会被子 Agent 的返回结果占用**

即使我们只返回了结构化 JSON，这个 JSON 体量也可能很大。如果子 Agent 需要回传长文本（例如全文翻译、长链分析），建议额外实现一个可引用的存储机制（如 artifact / 返回一个 file token），主会话里只放一个摘要和引用链接。

## 可复用建议

- **封装一个 subAgent runner 工具工厂**：接收 system prompt、工具列表、超时、输出 schema，自动完成子 session 的创建、运行、解析、错误兜底。
- **输出约束用 JSON Schema 住**：不要寄希望于 prompt 建议，能用格式化参数就用，OpenClaw 的 tool call 支持 structured output 更可靠。
- **监控与日志**：给子 session 的 token 消耗、耗时、失败次数打 log，后续优化 prompt 和模型选择有依可循。
- **可选的混合模式**：某些非关键场景，判断子 Agent 消息长度 < 500 字符时，也可以直接 append 回主会话，省去解析成本。但要加一个标志位，明确标注“以下内容来自子Agent”。
- **考虑流式场景的体验**：如果你的主会话要流式输出给前端，子 Agent 的阻塞调用会卡住整个流。建议子 Agent 调用异步化或者在流中展示一个“子任务进行中”的状态占位。

## 总结

OpenClaw 里 session 隔离的本质是，把子 Agent 当作一个“计算单元”，只把计算结果交回主会话，而不是把计算过程全部摊到对话里。实现要点就是三步：建子 session、给严格任务、只抽结论。踩坑集中在工具上下文丢失、输出格式不可靠、资源未清理这几个问题上。只要把这些包装成一个可复用的 runner，你就能放心地在自动化流程里大量使用子 Agent，而不用担心主会话被污染。

---

