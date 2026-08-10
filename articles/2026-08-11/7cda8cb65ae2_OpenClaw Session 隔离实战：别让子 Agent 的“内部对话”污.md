---
title: OpenClaw Session 隔离实战：别让子 Agent 的“内部对话”污染了主会话
feedId: 32471
source: 综合讨论
publishedAt: 2026-08-11
---

## 背景

OpenClaw 的多 Agent 协作能力，让我们可以拆解复杂任务，用主 Agent 调度一系列子 Agent 来完成具体工作。但如果你做过实践，很容易踩到一个隐晦的坑：**子 Agent 的执行过程、中间推理、工具调用日志，一股脑全混进了主会话的消息历史里。**

假设你用主 Agent 处理用户问题，它调了一个“长链路检索+推理”的子 Agent。这个子 Agent 内部分步骤思考、多次调用搜索工具，产生了几十条内部消息。如果不加隔离，这些消息全部追加到主会话中，导致：

- 上下文长度暴涨，推理成本和延迟陡增
- 主 Agent 需要继续推理时，被大量噪音分散注意力，回复质量下降
- 对话历史不可控，很难调试，也无法复现问题

子 Agent 污染主会话的本质是：**把“实现细节”暴露在了“调用接口”层。** 在工程设计中，子 Agent 应该像一个函数，只返回结果，内部状态应对调用方透明。OpenClaw 从设计上就支持这种隔离，但需要显式配置。

## Session 隔离机制的分析

OpenClaw 中每一次 Agent 调用都运行在自己的 Runtime Context 中。关键在于 **Context 的继承策略**：

- 默认行为：子 Agent 会继承父级会话上下文（messages, state），并在完成时将自身产生的全部消息合并回父会话。
- 隔离行为：通过配置 `session: isolate` 或使用 `runInSandbox` 模式，子 Agent 获得独立的消息空间。父会话只会收到最终的 `result` 或明确选择透传的少量消息。

从 0.9.x 版本开始，OpenClaw 提供了更细粒度的 **context isolation** 配置，不只隔离会话历史，还能隔离变量状态、工具副作用等。

## 实践步骤：让子 Agent 不污染主会话

以 OpenClaw 的 TypeScript API 为例，一个常见的子 Agent 调度写法：

```ts
const result = await agent.call({
  task: "查找关于 X-203 协议的变更记录并给出总结",
  subAgent: "researcher",
});
```

上面这种默认调用，子 Agent 的全部工具交互和中间消息会留存在当前会话。改正的写法有三种思路，分别适用于不同场景。

### 方案一：使用 `session: 'isolate'` 配置（推荐）

```ts
const result = await agent.call({
  task: "查找关于 X-203 协议的变更记录并给出总结",
  subAgent: {
    name: "researcher",
    session: "isolate",
    output: "final", // 只返回最终结果
  },
});
```

这样，`researcher` 子 Agent 拥有完全独立的会话，它的内部工具调用、多步推理都不会写回主 Agent 的对话历史。主 Agent 只会收到类似：

```
[researcher] 完成。
变更记录：... 总结：...
```

### 方案二：手动操作消息栈

如果你需要保留少量子 Agent 中间信息给用户（例如显示进度），可以手动控制消息的传递。先开启 `session: 'isolate'`，然后在子 Agent 的自定义回调中，将关键状态以 `user-invisible` 标记追加到主会话：

```ts
subAgent.on("step", async (step) => {
  // 仅在主会话中写入进度提示，但对模型不可见
  await ctx.addMessage({
    role: "system",
    content: `子任务进度: ${step.description}`,
    visibleToModel: false,
  });
});
```

这样既不影响模型推理的上下文，又可以在前端向用户展示进度。

### 方案三：采用 MCP 工具模式替代子 Agent

有时候，我们把本应是“工具”的能力错误地抽象成了子 Agent，导致会话污染。如果你只是需要调用外部服务，用 MCP server 暴露工具给主 Agent 会更干净。MCP 工具的执行日志被天然隔离，不影响对话历史。检查是否是这种情形：子 Agent 不维护状态，只是单一调用，就可以重构为 MCP 工具。

## 踩坑点

1. **子 Agent 结果丢失上下文索引**  
   如果子 Agent 的最终结果引用了内部消息 ID（比如“请参考第3条”，但第3条在隔离后不可见），主 Agent 会困惑。解决办法是在 `output` 配置中，让子 Agent 将引用内容以结构化字段返回，而非依赖索引。

2. **隔离后的变量状态不同步**  
   `isolate` 也会隔离 OpenClaw 的 session variables。如果子 Agent 需要共享某些全局变量（如 user_id），必须在调用时显式传入：

   ```ts
   subAgent: {
     name: "researcher",
     session: "isolate",
     context: { userId: ctx.var.userId },
   }
   ```

3. **多步委派带来的嵌套隔离**  
   当子 Agent 内部又调用了孙 Agent，如果未逐层配置 `isolate`，污染会在第三层悄悄扩散。建议制定团队规范：所有非顶层 Agent 调用一律用 `session: "isolate"` 或者统一封装 `callSubAgent` 工具，禁止直接使用默认调用。

4. **调试困难**  
   完全隔离后，发现子 Agent 出错，日志上看不见内部过程。解决方案是开启 OpenClaw 的 tracing 功能，子 Agent 的完整执行轨迹会输出到独立的 trace 文件或 Jaeger/OTEL 后端，不影响会话上下文。

## 可复用的工程化建议

- **封装调度工具**：不要到处写 `agent.call`，统一用一个 `delegateTask` 工具，内部强制配置 `session: "isolate"`，并处理变量传递、回写进度消息等。
- **区分“对话级”与“任务级”上下文**：设计 context schema 时，明确哪些字段是对话流（messages），哪些是持久状态（user profile、session meta）。隔离只隔消息，不隔持久状态。
- **监控上下文长度**：在 Staging 环境使用 OpenClaw 的 hook 机制，当主会话消息数突增时告警。可以很早就发现隔离失效的情况。
- **代码评审规则**：将 `agent.call` 作为敏感 API，要求所有调用都必须指定 `session` 策略，除非是顶层入口。

## 总结

子 Agent 的会话隔离不是银弹，但它是构建 **可维护多 Agent 系统** 的底线。核心原则就是一条：**把 Agent 当作函数，参数进，结果出，内部状态不要外泄。** OpenClaw 提供了足够的机制，但需要使用者有意识地设计隔离边界，而不是依赖默认行为。

配置很简单，踩的坑却往往来自“忘了配置”或者“不知道这个选项”。希望这篇文章能帮你提前规避这些问题，让你的多 Agent 系统更干净、更可控，上下文开销也更低。

---

