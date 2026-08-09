---
title: Agent 的 Subagent 编排实战：让多个 AI 并行做事，而不是排队
feedId: 32280
source: 综合讨论
publishedAt: 2026-08-09
---

## 背景：什么时候你需要多个 Agent 一起工作

单个 Agent 加再加一堆工具，是很多人的第一反应，但当任务量从「处理一封邮件」变成「分析 20 个 GitHub 仓库的 README 并生成审计摘要」时，串行执行的时间成本和上下文窗口压力会迅速暴露。

我在一个内部知识库巡检自动化里碰到真实场景：每晚需要调用搜索引擎获取 50+ 个技术关键词的最新动态，再用 RAG 对比内部文档，最后生成差异报告。如果让一个 Agent 逐个处理，单次需要 40 分钟，时不时还会因为上下文过长而丢信息。

于是方案转为 **Supervisor + Worker 模式的并行 Subagent 编排**：一个轻量编排 Agent 负责拆解任务，多个专职 Worker Agent 同时干活，最后聚合结果。本文在 OpenClaw 体系下完整跑通这套流程，分享工程实践和真实踩坑点。

## 问题：并行化不难，工程化才难

给每个子任务开一个 Agent 实例并不复杂，真正会出问题的地方在于：

1. **上下文隔离**：Worker 之间绝对不能共享记忆，否则相互污染，幻觉雪崩。
2. **结果可解析**：Worker 输出的不是给人看的总结，而是必须被下一个步骤程序化消费的结构化数据。
3. **并发约束**：API 速率限制、LLM 并发上限、Worker 互相抢占资源。
4. **容错与超时**：30% 的 Worker 可能失败或超时，不能拖垮整个任务。

这些点如果不提前设计，并行跑起来只会撞得更惨。

## 做法：基于 MCP 工具的并行 Worker 编排

OpenClaw 允许通过 MCP（Model Context Protocol）将 Agent 能力封装为标准服务，因此我们把每个 Worker Agent 暴露为一个 MCP 工具，编排 Agent 通过并行调用这些工具来实现 Subagent 并发。

### 第 1 步：定义专职 Worker Agent

每个 Worker 有独立 system prompt、独立工具集，且 `session_policy` 设置为 `isolated`（每次调用创建一个全新上下文）。以搜索 Agent 为例：

```yaml
# worker_search.yaml
name: search-worker
system: 你是一个搜索分析器，输出严格 JSON，只返回搜索结果的结构化摘要。
tools: [web_search, url_fetch]
session_policy: isolated
```

结构上刻意让 Worker “不思考太多”，只完成原子任务。

### 第 2 步：将 Worker 封装为 MCP 工具

在 OpenClaw 中通过 MCP Plugin 给每个 Worker 注册为一个工具，对外暴露一个标准接口：

```
tool_name: search_worker
description: 并行调用搜索 Worker，输入 json { "query": "...", "id": "task_001" }
```

这一步的关键是向编排 Agent 提供一个 **无需关心 Worker 内部细节** 的抽象工具，调用方式完全一致。

### 第 3 步：编排 Agent 设计

编排 Agent 负责两件事：拆分任务和并行调度。

我们在其 system prompt 里明确要求它分两步走：

1. 根据输入生成 N 个子任务描述（固定 JSON 数组）。
2. 调用 `parallel_dispatch` 工具，一次性提交全部子任务。

`parallel_dispatch` 是我们写的自定义工具，内部逻辑很直接：

- 接收子任务列表。
- 使用 `asyncio.gather` 并行调用对应的 MCP Worker 工具。
- 收集所有结果，失败的任务记录错误但不中断整体。
- 返回统一的聚合 JSON。

伪代码片段：

```python
async def parallel_dispatch(subtasks: list[dict]) -> dict:
    tasks = []
    for st in subtasks:
        tasks.append(call_worker_tool(st["tool"], st["input"], timeout=120))
    results = await asyncio.gather(*tasks, return_exceptions=True)
    return aggregate_results(subtasks, results)
```

这样，编排 Agent 只需一次调用，就可以驱动几十个 Worker 并行工作。

### 第 4 步：结构化输出与容错

每个 Worker 的输出必须强类型约束。我们使用 Pydantic 模型校验，不合规的输出触发一次自动重试（附带更严厉的格式提醒）。如果重试仍旧失败，该任务标记为 `skipped`，不阻塞整体。

## 踩坑记录

**坑 1：Worker 上下文残留**  
即使配置了 session 隔离，某些 MCP 实现会默认复用连接，导致前一个任务的 tool 响应残留在连接缓冲里。解决方法是在 Worker 端加上一个空 `reset_context` 调用，在每次正式执行前刷新。

**坑 2：API Rate Limit 雪崩**  
30 个 Worker 同时向同一个 LLM 后端发起请求，直接触发 429。引入 `asyncio.Semaphore`，限制最大并发数为 5，并配合指数退避重试，吞吐量下降不到 20%，但稳定性极大提升。

**坑 3：长任务耗尽编排 Agent 的上下文**  
聚合大量结果塞回给编排 Agent 时，很容易超出 token 限制。实践是将汇总分为两层：先由每个 Worker 输出摘要级结果，编排 Agent 只看摘要，详细结果另存文件，让下游工具按需加载。

**坑 4：Worker 输出幻觉成自然语言**  
尽管要求 JSON，某些 LLM 还是会输出“好的，这是结果：{...}”这样不纯的文本。我们增加了解析净化层，用正则提取 JSON 块，成功率从 80% 提升到 98%。

## 可复用建议

1. **无状态 Worker**：一律 `isolated` 上下文，不保存任何 session 信息。
2. **单 Worker 单职责**：越简单越好，prompt 控制在 3 句以内。
3. **统一追踪**：每个子任务携带 `correlation_id`，贯穿日志，方便排错。
4. **渐进压测**：先跑 2 个 Worker，观察延迟和速率限制，再扩大到目标并发度。
5. **准备 fallback**：关键子任务失败时，提供降级逻辑（比如使用缓存数据）。

## 总结

并行 Subagent 编排并不是银弹，但在任务可天然拆分的场景里，它能在几分钟内完成串行需要一小时的工作。核心在于把“工程约束”提前设计，而不是出了问题再打补丁。OpenClaw 的 MCP 生态让 Worker 封装和调用变得相对干净，配合简单的 Semaphore、超时和结构化解析，就能搭出生产可用的多 Agent 并行流水线。

并行之后，下一个有意思的方向是动态 DAG 调度——让 Agent 根据中间结果决定接下来唤醒哪些 Worker，那才是更复杂的编排未来。

---

