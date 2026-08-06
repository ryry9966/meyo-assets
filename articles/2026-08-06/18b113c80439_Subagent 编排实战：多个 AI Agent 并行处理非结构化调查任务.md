---
title: Subagent 编排实战：多个 AI Agent 并行处理非结构化调查任务
feedId: 31871
source: 综合讨论
publishedAt: 2026-08-06
---

## 背景

在自动化流程里，经常遇到一个典型场景：主 Agent 需要收集、校对或汇总多个独立信息源的数据，比如同时查询 5 个天气城市的实时数据、并行抓取 10 个竞品页面的结构化摘要、或对一份长文档的不同章节分别做审查。

如果让单个 Agent 串行处理，耗时 = sum(每个子任务时间)。但更高效的方式是，将任务拆解为独立的子任务，分配给多个 subagent 并行执行，再用主 Agent 汇总结果。这就是 **subagent 编排** 的核心价值。

在 OpenClaw 这类支持 MCP 工具和插件编排的框架下，实现 subagent 编排并不困难。但这背后隐藏着并发控制、错误隔离和结果聚合的工程问题。本篇文章基于我们团队最近一个真实案例（并行调查多个 SaaS 竞品的页面功能），梳理一套可复用的实践。

## 问题定义

需求：输入 8 个 SaaS 产品官网 URL，要求每个产品提取关键信息：核心功能列表、定价模型、主要集成。最终输出一份对比表格。

约束：所有页面需在 30 秒内完成调查，超时直接标记“未获取”，不能阻塞整体流程；部分页面可能反爬或返回非结构化内容，不能污染最终汇总；成本可控（避免并行数爆炸）

如果使用单 Agent 顺序抓取，8 页总耗时超过 2 分钟。于是我们引入 **1 个主调度 Agent + N 个 subagent (N=8)** 的编排模式。

## 具体做法

我们的工具链：OpenClaw (Agent 主程) + 2 个 MCP 服务器 (一个负责 web fetch，一个负责 resume 解析) + 一个简单的任务队列与限流器。

### 1. 拆解任务并定义 subagent 的输入输出契约

主 Agent 接收产品列表，生成结构化的任务描述。每个 subagent 只处理一个 URL，且必须返回固定的 JSON schema：

```json
{
  "url": "...",
  "status": "success|timeout|error",
  "data": {
    "features": ["...", "..."],
    "pricing": "freemium|subscription|...",
    "integrations": ["..."]
  }
}
```

这一步的关键是 **契约先行**，否则后续汇总时格式不一致会消耗大量主 Agent 的 token 做修正。

### 2. 创建 subagent 实例并并行调度

OpenClaw 中可以动态创建子 Agent，每个子 Agent 配置自己的工具集（比如只允许使用 `web_fetch` 和 `parse_markdown` 工具）。我们用 asyncio 并发执行：

```python
async def run_subagent(url):
    try:
        sub = await openclaw.create_subagent(
            prompt=f"Investigate {url} ...",
            tools=["web_fetch", "parse_markdown"],
            output_schema=INVESTIGATION_SCHEMA,
            timeout=25  # 单个子任务超时
        )
        return await sub.run()
    except Exception as e:
        return {"url": url, "status": "error", "data": None}
```

然后使用 `asyncio.gather(*[run_subagent(u) for u in urls], return_exceptions=True)` 并行启动 8 个 subagent。注意 `return_exceptions=True` 是为了不让单个子任务异常导致全流程崩溃。

### 3. 并发控制与限流

直接对 8 个 URL 同时发起请求容易触发目标站点的反爬，也容易让底层 API （如果 subagent 仍依赖 LLM 推理）遇到 rate limit。我们引入 Semaphore 限制实际并发数：

```python
sem = asyncio.Semaphore(4)   # 最多4个并发

async def rate_limited_run(url):
    async with sem:
        return await run_subagent(url)
```

这样 8 个任务分两批执行，并发度可控，既满足 30 秒总时长，又避免被 blocked。

### 4. 结果聚合

主 Agent 等待所有 subagent 返回，然后根据 status 分类处理。对于 `success` 的结果，直接抽取字段合并成表格；对于 `timeout` 或 `error`，补上占位信息，并在最终报告中标注“数据缺失”。

我们用了一个小技巧：聚合前先做一轮 “schema validation”，过滤掉不符合输出契约的返回值，避免坏数据进入表格。这步由主 Agent 调用一个简单的 Python 函数完成，而非用 LLM 做判断，成本更低。

## 踩坑点

1. **子 Agent 的上下文膨胀**  
   如果 subagent 的 system prompt 过长或携带过多无关工具，并行时会迅速耗尽上下文窗口。建议只为子 Agent 配置必需工具，提示词精炼。

2. **无差别并行导致 API Rate Limit**  
   无论是 LLM API 还是外部网页请求，大量并发都会触发限流。必须根据实际限流阈值设置并发上限，最好在测试中摸清“安全并发数”。

3. **子任务超时设置不当**  
   设置全局 timeout 容易误伤慢任务，不设置又会永久阻塞。我们为 subagent 设置 25 秒超时，同时主流程设 35 秒整体超时，一旦整体超时立即取消未完成的子任务，避免悬空。

4. **错误传播与噪声**  
   一个 subagent 崩溃若没有 `return_exceptions`，会直接中断 gather。此外，错误信息如果不结构化，汇总时会混入无意义堆栈。务必在 subagent 内捕获异常并返回符合 schema 的错误对象。

## 可复用建议

- **适用场景**：任务之间无依赖、输出结构固定、单任务耗时适中（几秒到几十秒）。如果需要任务间通信或顺序依赖，不适合简单并行。
- **并发数选择**：先单任务测平均耗时，根据总时限反推最大并行数，再乘以 0.7 作为安全上限。
- **契约测试**：在正式跑全量任务前，先用 2-3 个样本验证 subagent 的输出 schema 稳定性。
- **监控与降级**：记录每个 subagent 的运行时长和状态，一旦失败率超过阈值，可自动切换为串行模式保底。
- **成本控制**：并行数越大瞬时 token 消耗越高，注意给主 Agent 的聚合 prompt 设置 max_tokens 限制，避免汇总表格时无意义的长篇说明。

## 总结

Subagent 编排不是银弹，但确实能让许多调查、校验类任务的时间从“串行累加”变成“并行重叠”。在 OpenClaw 这类框架下，实现难度不高，关键是把工程细节处理好：契约先行、隔离异常、控制并发、结构化输出。当你下次需要同时查询 10 个数据源或审查多个文件时，不妨试试这个方法，并一定为“部分失败”做好预案。

---

