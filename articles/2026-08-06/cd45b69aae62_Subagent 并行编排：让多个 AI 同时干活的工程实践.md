---
title: Subagent 并行编排：让多个 AI 同时干活的工程实践
feedId: 31778
source: 综合讨论
publishedAt: 2026-08-06
---

## 背景：从串行到并行

在 Agent 自动化流程里，单 Agent 串行调用工具的模式很常见：先读文件 → 再查 API → 再汇总 → 再发邮件。很多步骤之间其实没有数据依赖，完全可以同时进行，但大多数实现是一步一步等，整体耗时等于各步骤之和。

我们遇到一个实际的自动化场景：需要对 10 个独立数据源分别做质量检查并生成报告。如果让一个 Agent 逐个处理，总耗时超过 3 分钟；把每个数据源检查封装成独立的 subagent，并行执行后，整体耗时缩短到 30 秒内。

这篇文章不聊概念，直接讲在 OpenClaw 生态下如何安全地让多个 AI subagent 并行做事，以及踩过的坑。

## 问题拆解

并行编排不止是 `asyncio.gather` 一下那么简单。工程上需要解这几个问题：

- **子任务独立性判定**：哪些步骤可以并行，哪些必须串行？这件事得交给主 Agent 做决策，并且要在 prompt 里明确约束。
- **资源和速率控制**：多个 subagent 会同时消耗 LLM API 额度，非常容易触发 rate limit 或 cost 暴涨。
- **上下文隔离**：subagent 之间不能共享会话状态，否则容易出现幻觉交叉感染。
- **健壮性**：一个子任务挂了，不能拖垮其他子任务，也不能让主流程无声失败。

## 做法/步骤（以 OpenClaw 为例）

OpenClaw 不提供内置的 subagent 并行执行器，但它的工具调用机制完全支持异步并发。我们可以把每个 subagent 封装成一个无状态的 `FunctionTool`，由主 Agent 在一次回复中触发多个工具调用，然后用异步执行层并发跑这些调用。

### 1. 定义无状态 subagent 工具

每个 subagent 必须是纯函数式的：输入明确的参数，返回结构化结果，不读不写全局状态。例如：

```python
async def check_datasource(name: str, config: dict) -> dict:
    # 内部可能再次调用 LLM 或 HTTP，但上下文完全隔离
    ...
```

用 `@tool` 装饰后注册到 OpenClaw Agent 的工具集里。

### 2. 主 Agent 的 system prompt 约束

关键一步：必须让主 Agent 知道哪些工具可以并行调用，并明确规则。我们加的约束大致是：

> When you need to invoke multiple tools that have no data dependency on each other, you MUST emit all their tool calls in a single response. The execution environment will run them concurrently. Never split independent calls across multiple turns.

实测不加这句话，模型容易走回串行老路，尤其是 GPT-4o 这类模型会倾向于一次只调用一个工具。

### 3. 并发执行层

在工具路由层（`ToolExecutor` 或等价位置），拦截同一轮对话产生的多个工具调用，使用 `asyncio.gather` 并发执行，并加入三个控制机制：

- **Semaphore 并发限制**：避免瞬间打爆 API。我们通常设上限为 5。
- **超时兜底**：`asyncio.wait_for`，单个调用超时可返回降级结果而不是无限等。
- **异常隔离**：单个 subagent 失败不影响其他，返回 `{"error": str(e)}` 而不是向上抛异常。

代码示意：

```python
sem = asyncio.Semaphore(5)

async def run_one(call):
    async with sem:
        try:
            return await asyncio.wait_for(tool.invoke(call), timeout=60)
        except Exception as e:
            return {"error": str(e)}

results = await asyncio.gather(*[run_one(c) for c in tool_calls])
```

### 4. 结果聚合与主 Agent 继续推理

并行结果按工具调用顺序收集后，原样注入回对话，主 Agent 拿到所有子任务结果再做决策。这样既保持了推理的连贯性，又避免了反复问询。

## 踩坑实录

- **Rate limit 风暴**  
  某次我们把并发上限设成 10，30 秒内 4 个 key 全被限流。最终方案是 Semaphore + 指数退避重试，并且对超过 429 的工具结果做退避标签，主 Agent 不再重试该子任务。

- **主 Agent 假装没有依赖**  
  我们见过模型把“先查用户的账户 ID，再查账户余额”这种有明显串行依赖的任务强行并行，导致第二个调用拿到的是凭空捏造的账户 ID 去查询。解决办法是 prompt 里增加大量 few-shot 示例，并强调“If you don’t have the result of tool A, you cannot call tool B with it”。必要时可在执行层加一个轻量的依赖检测。

- **上下文长度膨胀**  
  并行结果全部写回历史，对话窗口很容易爆炸。我们做了结果摘要化：subagent 只返回关键数据，不返回冗长日志；主 Agent 拿到结果后做一次压缩再进下一步。

- **成本控制**  
  并行意味着同时烧 token，需要给每个 subagent 设置 max_tokens 上限，避免某个子任务无节制输出。

## 可复用建议

- **封装一个 `ParallelToolExecutor`**：把 Semaphore、超时、异常隔离、结果标准化全装进去，外部只需传入工具调用列表。  
- **给每个 subagent 加“预算”**：包括 token 上限、最长执行时间、最大重试次数，执行器统一遵守。  
- **保持主 Agent 无状态**：subagent 不该修改主 Agent 的 memory，主 Agent 仅根据工具返回结果推进。  
- **日志和痕迹**：每个并行执行的整体耗时、各工具返回值长度、错误原因都要记录，方便后期调优。  
- **渐进式开启并行**：先在简单场景验证，用 `parallel_degree` 参数控制，不要一口气全量切换。

## 总结

多个 AI 并行干活不是花活儿，而是减少端到端延迟的实在手段。但工程上必须做好资源控制、依赖判定和失败隔离，否则并行带来的问题比串行还多。对于 OpenClaw 用户，利用好异步工具调用和执行层的并发封装，完全可以低成本实现可靠的 subagent 并行编排。

---

