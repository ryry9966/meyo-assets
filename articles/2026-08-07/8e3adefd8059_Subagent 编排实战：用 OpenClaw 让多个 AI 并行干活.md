---
title: Subagent 编排实战：用 OpenClaw 让多个 AI 并行干活
feedId: 32021
source: 综合讨论
publishedAt: 2026-08-07
---

## 背景

在 OpenClaw 的日常使用中，单个 Agent 串行执行任务是最直观的模式：查资料→分析→写结论，一步步走完。但当任务涉及多个独立信息源时，比如同时拉取三个 REST API 的数据、对比五个网页的内容、或者并行查询数据库与文件系统，串行模式的耗时就开始变得不可接受。

OpenClaw 的 subagent 机制允许我们将主 Agent 的任务拆分为多个子代理，分别绑定不同的工具和 MCP 服务器，然后并发执行。利用异步 I/O 和 MCP 的多连接能力，我们可以实现类似“分派三个助手同时干活，最后汇总”的效果。本文记录这一过程的工程化实践，包括拆分策略、编排代码、踩坑点和可复用的判断标准。

## 问题：从串行到并行的工程挑战

单 Agent 串行执行之所以简单，是因为上下文是线性的，错误处理也直观：一步失败，后面停止。引入并行 subagent 后，会立刻面临几个新问题：

1. **任务无依赖拆分**：并非所有步骤都可并行，只有那些不相互依赖的子任务才适合并发出击。
2. **结果聚合与冲突**：多个 subagent 可能操作同一资源（如写同一个本地文件），需要设计避免竞态的机制。
3. **资源与限流**：MCP 服务器可能有单连接限制，LLM 调用速率有限，Token 消耗成倍增长。
4. **部分失败场景**：一个子任务超时，是否要放弃其他已经运行的任务？错误如何回传？

## 实践步骤

### 1. 设计并行子任务

先梳理主流程，识别出没有先后依赖的“独立工作单元”。例如，我们的需求是：查询天气、分析今日新闻、获取股票数据，三者之间互不依赖，就可以并行。

在 OpenClaw 中，为每个单元定义一个 subagent，并给其专属的 system prompt 和工具权限：

```python
# 示例：定义三个 subagent 的函数
async def weather_subagent():
    return await openclaw.create_subagent(
        name="weather",
        system_prompt="You fetch current weather for a given city.",
        mcp_servers=["weather-mcp"],
    ).run("Get weather for Beijing")

async def news_subagent():
    return await openclaw.create_subagent(
        name="news",
        system_prompt="Summarize today's top tech news.",
        mcp_servers=["web-search-mcp"],
    ).run()

async def stock_subagent():
    return await openclaw.create_subagent(
        name="stock",
        system_prompt="Return AAPL and GOOG stock prices.",
        mcp_servers=["stock-mcp"],
    ).run()
```

### 2. 并行执行与结果汇总

使用 `asyncio.gather` 或 OpenClaw 内置的并行调度器（如果有）同时启动：

```python
import asyncio

results = await asyncio.gather(
    weather_subagent(),
    news_subagent(),
    stock_subagent(),
    return_exceptions=True   # 关键：捕获异常而不中断其他任务
)

# 将结果交给主 Agent 汇总
final_report = await main_agent.run(
    "Combine the following results into a concise report:
"
    f"Weather: {results[0]}
News: {results[1]}
Stock: {results[2]}"
)
```

`return_exceptions=True` 允许我们按需处理部分失败：如果某个子任务返回异常，主 Agent 可以决定是忽略还是重试。

### 3. 超时与资源控制

为每个子任务设定超时，防止一个慢响应拖死整个批次：

```python
async def run_with_timeout(coro, timeout=30):
    try:
        return await asyncio.wait_for(coro, timeout=timeout)
    except asyncio.TimeoutError:
        return {"error": "subagent timed out"}
```

同时注意 MCP 连接数。如果某个 MCP 服务器只支持单 Session，并发调用会导致 `connection refused`。对策是使用信号量限制并发度，或为每个 subagent 分配不同的 MCP 实例。

## 踩坑点

### 上下文隔离 ≠ 资源隔离
Subagent 的上下文独立，但它们共享宿主环境（文件系统、环境变量）。我曾让两个 subagent 并行写同一个报告文件，结果最后文件内容交错，只能重跑。解决方案：让每个 subagent 返回结构化 JSON，由主 Agent 负责写入，或者使用临时文件 + 唯一命名。

### Token 账单爆炸
并行 4 个 subagent，每个消耗 2000 token，加上汇总 prompt，总 token 可能轻松破万。务必在开发阶段通过 `max_tokens` 限制每个子任务的输出长度，并在 prompt 中明确要求“回答不超过 200 字”。

### 结果顺序混乱
`asyncio.gather` 返回值的顺序与传入协程的顺序一致，但如果用 `asyncio.as_completed`，则顺序不定。在汇总时需要能够按任务标识匹配结果，因此子任务输出最好带上任务名或 ID。

### MCP 连接池问题
目前部分 MCP 实现并非连接池友好，短时间大量重连可能被目标服务限流。建议对频繁调用的 MCP 启用长连接复用，或通过 agent 分组来降低瞬时并发。

## 可复用建议

**适用场景**：
- 数据聚合：多 API、多数据库、多网页抓取
- 分块处理：大文档分段分析、多文件代码审查
- A/B 策略：同时让两个子 Agent 用不同方法解题，再选择最佳结果

**避免过度拆分**：如果单个子任务本身就很轻量（如一次简单的 SQL 查询），并行带来的调度开销可能大于收益。衡量标准：子任务运行时间 > 1 秒才值得并发出。

**监控与日志**：
OpenClaw 的事件系统可以记录每个 subagent 的启动、结束和异常。建议在主 Agent 中加入简短的统计输出，例如打印每个子任务的耗时和 token

**封装通用并行执行器**：
把超时、异常处理、结果组装抽成一个工具函数，主 Agent 只需传入任务列表即可。这样可以复用工程经验，也方便后续升级到更高级的编排模式（如流水线、DAG）。

## 总结

Subagent 的并行编排是 Agent 从“帮手”升级为“团队”的关键一步。它让 AI 能够像多线程程序一样同时推进多个独立任务，显著压缩了端到端延迟。但并行不是银弹：任务拆分、异常隔离、资源控制和成本监控不可或缺。只有在适当的场景下，辅以严谨的工程手段，才能把多 Agent 并行的价值真正落进生产。

---

