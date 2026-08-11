---
title: Subagent 并行编排：在 OpenClaw 中让多个 AI 同时干活的工程实践
feedId: 32694
source: 综合讨论
publishedAt: 2026-08-12
---

## 背景

单个 AI Agent 在处理复杂任务时，通常会按顺序逐项完成子任务——分析需求、查阅资料、生成内容、调用工具……这种单线程模式在子任务之间存在数据依赖时是合理的，但在大量**可独立执行的子任务**面前，串行就成了瓶颈。例如：

- 需要同时爬取 5 个新闻站点的头条摘要；
- 同时审查多个代码仓库中的依赖漏洞；
- 同时翻译一份多语言文档的若干章节。

这些子任务互相无依赖，结果只需简单汇聚即可形成最终输出。如果能让多个 subagent 并行干活，整体响应时间将从“求和”变为“取最大值”。

本文围绕 OpenClaw 的 Agent/MCP 工具链，给出一种**可落地的 subagent 并行编排方案**，包含工程化的拆解、实现、踩坑与可复用建议。

## 问题拆解

要在 OpenClaw 中实现并行 subagent，核心要解决三个问题：

1. **上下文隔离**：每个 subagent 需要独立的对话上下文，不能互相污染指令或历史。
2. **工具/资源竞争**：多个 subagent 可能同时调用同一个 MCP 工具（如数据库查询、网页抓取），容易触发限流或锁冲突。
3. **结果汇聚与容错**：部分 subagent 可能失败，需要保证整体任务不被拖垮，且能给出部分成功的结果。

OpenClaw 提供了 Agent 基类与 MCP 工具集成，但默认的一次 `agent.run()` 并不天然支持“一对多”分发。我们需要在编排层自己实现并行控制。

## 实现步骤

以“同时抓取多个数据源并汇总”为场景，实现一个 `ParallelCrawler` 主 Agent，它接收数据源列表，调度 subagent 并行抓取，最后汇总。

### 1. 设计 Subagent

为每个抓取任务定义一个轻量 subagent，功能极其单一：接收一个 URL，使用网页抓取工具获取内容，返回结构化 JSON。

```python
from openclaw import Agent

class CrawlerSubAgent(Agent):
    def __init__(self, tools):
        super().__init__(
            name="crawler_sub",
            system_message="你是一个网页抓取助手。只使用 fetch_url 工具，将返回的内容整理成 {title, summary, key_points} 格式返回。",
            tools=tools,
        )
```

关键点：system_message 必须严格限定子任务边界，减少无关输出，降低 token 消耗。

### 2. 主 Agent 的并行调度

主 Agent 使用 `asyncio.gather` 启动多个 subagent 实例，每个实例有独立的对话上下文，可共享同一组工具（但要小心资源竞争）。

```python
import asyncio

async def run_subagent(url, tools):
    agent = CrawlerSubAgent(tools)
    result = await agent.run(f"请抓取 {url} 的内容并返回结构化结果")
    return result

async def parallel_crawl(urls, tools):
    tasks = [run_subagent(url, tools) for url in urls]
    results = await asyncio.gather(*tasks, return_exceptions=True)
    return results
```

- `return_exceptions=True` 确保个别子任务失败不会中断全部；
- 每个 subagent 是独立的 Agent 实例，上下文天然隔离。

### 3. 结果汇聚

主 Agent 拿到并行结果后，进行二次整理：

```python
async def main_agent(urls, default_tools):
    raw_results = await parallel_crawl(urls, default_tools)
    # 解析成功结果，丢弃异常
    summaries = []
    for res in raw_results:
        if isinstance(res, Exception):
            summaries.append({"error": str(res)})
        else:
            # 假设 subagent 返回了 JSON 字符串
            summaries.append(json.loads(res))
    # 提交给主 Agent 进行最终分析
    main = Agent(name="summarizer", ...)
    final = await main.run(f"请根据以下抓取结果生成综合报告:\n{summaries}")
    return final
```

实际工程中，我会把汇聚逻辑也交给同一个主 Agent，让它根据原始子任务结果做智能合并，避免规则过多依赖代码硬解析。

## 踩坑点

### 1. MCP 工具并发瓶颈

多个 subagent 共享同一个 MCP Server（例如浏览器自动化工具 Playwright MCP），并发请求可能因 Server 内部单线程排队而超时，甚至因高频调用触发反爬。  
**解决**：针对容易限流的工具，为 subagent 池增加 `asyncio.Semaphore` 限制并发数，每批 3~5 个即可。

```python
sem = asyncio.Semaphore(3)
async def run_limited(url, tools):
    async with sem:
        return await run_subagent(url, tools)
```

### 2. 上下文窗口膨胀

每个 subagent 的 system_message 虽已精简，但工具描述、返回内容仍会占用大量 token。如果同时启动 10 个 subagent，总 token 消耗线性增长，可能触发 LLM 速率限制。  
**解决**：尽量选用 small 或 medium 尺寸的模型跑 subagent（如 GPT-4o-mini），只在主 Agent 汇总时使用高能力模型。

### 3. 结果格式不可控

Subagent 经常不按约定的 JSON 返回，夹杂自然语言解释，导致解析失败。  
**解决**：在 subagent 的 system_message 中增加“只输出合法 JSON，不要任何额外文字”的强约束，并使用 `response_format={"type": "json_object"}`（如果底层模型支持）。后端解析前做一次正则兜底。

### 4. 单点超时拖慢整体

一个 subagent 卡死会导致 `gather` 等待很久。  
**解决**：在每个子任务外包裹 `asyncio.wait_for(..., timeout=60)`，超时直接抛异常，由汇聚逻辑处理。

## 可复用建议

**封装成通用工具函数**  
将并行调度抽象为 `dispatch_subagents(task_list, agent_factory, concurrency=5, timeout=60)`，后续任何类似需求只需提供任务列表和 subagent 构造工厂。

**预先拆分任务**  
让主 Agent 先调用一次 LLM 将用户需求显式拆成子任务数组，再分发给并行 subagent。这样拆解与执行分离，更易调试。

**为 subagent 设计无状态工具**  
尽量让每个 subagent 调用幂等、无副作用的工具（只读查询、抓取），避免写入冲突。

**监控与降级**  
增加日志记录每个 subagent 的耗时与结果状态，当失败率超过阈值时，整体降级为串行或返回部分结果并告警。

## 总结

在 OpenClaw 生态中，利用 Agent 实例隔离与 Python 异步能力，可以低成本实现 subagent 并行编排。核心收益在于**将无依赖子任务的耗时从累加变为取最大**，极大提升复杂任务的响应速度。实践中需要重点控制工具并发、上下文开销和结果解析的鲁棒性。当你下次面临“一堆相似页面要分析”“多个仓库要检查”时，不妨试试这套轻量并行法。

---

