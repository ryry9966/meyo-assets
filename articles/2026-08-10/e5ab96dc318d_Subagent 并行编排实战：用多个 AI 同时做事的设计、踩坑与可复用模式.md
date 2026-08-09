---
title: Subagent 并行编排实战：用多个 AI 同时做事的设计、踩坑与可复用模式
feedId: 32322
source: 综合讨论
publishedAt: 2026-08-10
---

# Subagent 并行编排实战：用多个 AI 同时做事的设计、踩坑与可复用模式

## 1. 背景：从单 Agent 到并行编排的需求变迁

在 OpenClaw 生态中，单个 Agent 配合工具链（MCP 服务、Functions）已经能处理很多任务。但实际业务里经常遇到这类场景：

- 需要同时爬取多个数据源再做汇总分析；
- 批量代码仓库审查，每个仓库都需要独立的上下文；
- 多维度调研报告，比如市场、竞品、技术方案要并行调研；
- 用户请求明显可分拆，强行串行执行会超时或体验差。

如果只靠一个 Agent 依次处理，前面任务的长上下文、工具调用耗时都会拖慢整体进度。最直接的想法是：能不能创建多个“子 Agent”，把可并行的子任务分配出去，同时跑，最后汇总？

在 Agent 框架里，这就是 **subagent 编排**。本文不讨论通用概念，只聚焦在工程实现中实际踩过的坑和沉淀下来的模式。

## 2. 问题分析：并行编排的 3 个核心难点

从单 Agent 扩展到“主 Agent + 多个子 Agent 并发”，会立刻撞上三个拦路虎：

**a) 上下文隔离与管理**

每个子 Agent 需要独立的上下文窗口（Messages、Memory），避免互相污染。同时主 Agent 需要一种方式传递子任务参数，并回收结果，但不能把每个子任务的冗长执行过程全塞进主 Agent 上下文，否则一样会超限。

**b) 子任务粒度与依赖**

拆得太细，编排开销大、结果合并复杂；拆得太粗，并行度不够，又退化成串行。而且有些子任务可能有隐式依赖，比如第二个任务需要第一个任务的输出，这时就不能简单并行。

**c) 错误隔离与重试**

一个子任务挂掉是否影响全局？部分成功怎么办？如果主 Agent 等待所有子任务返回，某个长时间卡住会不会导致整体超时？

这些问题不是靠简单调用 `async`/`parallel` 就能解决，需要设计一套稳定的编排机制。

## 3. 做法与步骤

以下基于 OpenClaw 的 Tool / Agent 机制描述一个可落地的主-子并行模式。

### 3.1 抽象子任务的结构

定义统一的任务描述对象，让主 Agent 以结构化方式声明子任务：

```json
{
  "id": "repo-scan-1",
  "type": "code-review",
  "params": {
    "repo_url": "...",
    "focus": "security"
  },
  "output_format": "json"
}
```

关键点：每个子任务必须声明输出格式，便于主 Agent 后续解析。通常建议强制 JSON Schema，避免子 Agent 自由发挥。

### 3.2 主 Agent 的 Tool 设计：`dispatch_subagents`

在主 Agent 可调用的工具中，暴露一个 `dispatch_subagents` 函数，签名大致如下：

```python
async def dispatch_subagents(tasks: list[SubTask]) -> list[SubTaskResult]:
```

执行逻辑：
1. 为每个 `task` 创建独立 Agent 实例（可复用同一 Agent 类型，但上下文隔离）；
2. 将 `task.params` 和 `task.output_format` 注入子 Agent 的系统提示；
3. 使用 `asyncio.gather` 或线程池并发执行所有子 Agent；
4. 等待全部完成（或超时），收集返回值；
5. 将结果列表返回给主 Agent。

注意：这个 Tool 本身的执行对主 Agent 是“阻塞”的，主 Agent 等待工具返回结果后再继续推理。并发在工具内部完成。

### 3.3 子 Agent 的标准行为

子 Agent 的系统提示要极简化，只做一件事：根据输入参数执行任务，并严格按照 `output_format` 返回结果。禁止在系统提示里写“你可以继续问用户”、“处理多个任务”等泛化能力，那样反而容易产生幻觉或不必要对话。

示例系统提示：

```
You are a subagent. Your only job is to complete the given task.
Input parameters will be provided in the first user message.
You MUST output the result exactly in the specified JSON format.
Do not ask for clarifications. Do not output explanations outside the JSON.
```

### 3.4 超时与部分失败处理

在 `dispatch_subagents` 内部设置每个子任务的超时时间（如 120s）。超时未返回的任务，结果标记为 `timeout`，并返回主 Agent。主 Agent 根据返回状态决定下一步：忽略、重试还是报告给用户。

部分失败模式要提前设计。比如 3 个任务中 1 个失败，其他 2 个的结果仍然可用。这时子任务结果对象应包含 `status` 字段（`success`/`failed`/`timeout`），主 Agent 汇总时可根据策略处理。

## 4. 踩坑点

**坑 1：上下文膨胀**

主 Agent 调用 `dispatch_subagents` 时，传入的子任务列表本身占 tokens；返回的多个结果也会一次性塞进主 Agent 上下文。如果一个子任务返回了满屏 JSON（例如 5000 行的文件列表），主 Agent 后续推理很可能因上下文过载而劣化。

**解决**：采集结果时做摘要截断。例如超过 2000 字符的结果只保留关键统计，完整内容写入外部存储（文件、KV 存储），返回给主 Agent 的仅是一个引用路径。

**坑 2：子 Agent 的幻觉输出格式**

即便系统提示要求 JSON，某些模型在复杂任务下仍然可能输出 `{"result": "... (markdown here)"}` 或包裹解释文字。这会导致主 Agent JSON 解析失败。

**解决**：在子 Agent 外层包一个“输出格式校验器”。尝试解析 JSON，如果失败，将原始输出连同错误信息返回给主 Agent，而不是直接崩溃。同时可以在子 Agent 提示中加入“如果无法生成合法 JSON，请返回 `{"error": "..."}`”的兜底指令。

**坑 3：并发数控制缺失**

调用方可能传入 50 个子任务。如果不管控并发，会同时创建 50 个 Agent 实例并发送请求，触发 rate limit 或 OOM。

**解决**：使用信号量控制最大并发数（建议 5–10）。例如 `asyncio.Semaphore(max_concurrency)`，配合 `gather`，保证资源可控。

**坑 4：用户感知到的等待时间**

并行子任务后，用户看到的是一个总的等待时间，而不是逐步输出。主 Agent 在此期间无法返回中间进度。

**改善**：在 `dispatch_subagents` 内部使用流式进度回调（如 WebSocket 推送状态），主 Agent 的工具可以对外暴露进度。不过这需要框架支持，否则只能退而求其次，把总超时设置合理并告知用户。

## 5. 可复用建议

经过多次迭代，我固化了一些可复用的模式：

- **主 Agent 只负责任务拆分与结果决策**，不参与执行细节。这是保证可维护性的关键。
- **子 Agent 无状态、纯函数化**。输入参数 + 输出 JSON，不要让子 Agent 记忆历史。
- **统一的结果信封（Envelope）**：所有子任务返回 `{"status": "...", "data": {...}, "error": "..."}`，主 Agent 无需解析不同类型。
- **存储中间产物**：重要结果写入文件或数据库，避免上下文泄漏；同时为审计和重试提供依据。
- **设计重试策略但不盲目重试**：对 `timeout` 可重试一次，对明确 `failed` 的不要重试，直接报告。
- **在提示中给出示例输出**：显著降低格式错误率。

## 6. 总结

Subagent 并行编排不是银弹，但能显著提升多独立子任务的吞吐，并保持主 Agent 的规划能力不被执行细节淹没。重点在于把并发细节封装进可靠的 Tool 内部，主 Agent 仍以声明式方式工作。工程上只要解决好上下文隔离、输出格式校验、并发控制和优雅降级，这套模式就能在 OpenClaw 生态的日常自动化中稳定运行。

最终的效果是：原来需要 5 分钟串行跑完的 3 个调研任务，现在 2 分钟内完成，且主 Agent 拿到的是干净、结构化、可直接用于决策的结果。这才是 Agent 自动化该有的效率。

---

