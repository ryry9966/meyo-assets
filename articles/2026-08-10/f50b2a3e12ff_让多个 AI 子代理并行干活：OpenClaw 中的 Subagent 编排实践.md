---
title: 让多个 AI 子代理并行干活：OpenClaw 中的 Subagent 编排实践
feedId: 32394
source: 综合讨论
publishedAt: 2026-08-10
---

## 背景：当你只有一个Agent却要同时做多件事

在基于 OpenClaw 构建自动化 Agent 时，常见场景是单一主 Agent 按照顺序调用工具、思考、再调用，形成串行流水线。对于一个“收集三个数据源的舆情并汇总成报告”的任务，串行意味着依次请求 API，总耗时等于三个请求之和。更糟的是，如果其中某个任务需要持续监听或等待回调，主 Agent 会被完全阻塞。

另一个问题是上下文污染。让一个 Agent 同时处理“解析 PDF 表格”“爬取竞品价格”“生成 SQL 报表”，其 system prompt 会膨胀，且中间推理很容易互相干扰。为了解决这两个核心痛点——**并行执行与职责隔离**，OpenClaw 提供了 Subagent 机制，允许主 Agent 像调用函数一样启动多个独立的子 Agent，委托它们完成子任务，最后汇总结果。

本文将记录我们在生产环境中用 OpenClaw Subagent 实现多 AI 并行做事的完整过程、踩过的坑以及可复用的工程化建议。

## 问题定义

以“竞品价格监控”为例，需求是：
1. 同时抓取 3 个目标网站的商品价格；
2. 对每个网页的内容进行结构化提取（名称、价格、库存）；
3. 汇总成一张表格，如果任一网站不可用，标记为失败但不影响整体。

如果用单个 Agent 串行处理，总耗时 = 网络请求 + 解析时间之和；且若第一个网站无响应，超时会拖慢整个流程。我们希望通过 Subagent 并行处理，实现耗时接近最慢的那个任务，同时隔离失败。

## 实践步骤

### 1. 主 Agent 设计
主 Agent 只负责调度，不直接处理业务逻辑。在 OpenClaw 中，我们需要定义一个用于启动 subagent 的 tool，并在该 tool 内部创建并运行 SubAgent 实例。

核心代码片段（基于 OpenClaw 内部 SubAgent API，简化示例）：

```python
from openclaw import Agent, Tool, SubAgent

def run_parallel_scrapers(urls: list) -> dict:
    """启动多个爬取子代理，并发执行"""
    subagents = []
    for url in urls:
        sub = SubAgent(
            name=f"scraper-{url.replace('https://','')[:20]}",
            system_prompt=(
                "你是一个网页抓取助手。给定 URL，使用 fetch_page 工具获取内容，"
                "然后提取其中的商品名称、价格和库存状态，以 JSON 格式返回。"
                "如果失败，返回 {'error': '原因'}。"
            ),
            tools=[fetch_page_tool],
            max_iterations=5
        )
        subagents.append(sub)

    # 并行启动所有子代理，并等待所有完成
    results = {}
    import concurrent.futures
    with concurrent.futures.ThreadPoolExecutor(max_workers=5) as executor:
        future_to_url = {
            executor.submit(sub.run, f"抓取并解析 {url}"): url
            for sub, url in zip(subagents, urls)
        }
        for future in concurrent.futures.as_completed(future_to_url):
            url = future_to_url[future]
            try:
                outcome = future.result(timeout=60)
                results[url] = outcome
            except Exception as e:
                results[url] = {"error": str(e)}
    return results
```

主 Agent 将该函数注册为 Tool，当用户说“监控这三个网站的价格”，主 Agent 会调用 `run_parallel_scrapers`，拿到字典结果后继续汇总。这样主 Agent 的上下文只看得见最终 JSON，不会被中间爬取细节污染。

### 2. 子代理的提示词与工具链
每个 SubAgent 有独立的 system prompt 和工具集，必须设计得小而专注。赋予它过多能力（例如同时又让爬虫去写报告）只会增加出错概率和 token 消耗。我们为爬虫子代理只给两个工具：`fetch_page` 和一个 `extract_json` 工具。在 system prompt 中明确要求“只返回 JSON，不要多说任何话”，这样便于主 Agent 解析结果。

### 3. 并发控制与错误隔离
上面代码中的 `ThreadPoolExecutor` 是示意，实际 OpenClaw 内部可能有自己的调度器。关键要点是：
- **设置最大并发数**：我们限制了 5 个 worker，防止同时产生过多子代理导致上下文窗口或 API rate limit 爆掉。
- **为每个子代理设置独立超时**：`future.result(timeout=60)`，避免某个任务僵尸化。
- **异常捕获后返回结构化错误**：让主 Agent 看到失败信息，而不是整个流程崩溃。

## 踩坑记录

1. **子代理返回不可解析的内容**
   即使提示词要求返回纯 JSON，有时子代理还是会加上 markdown 代码块标记或多余解释。最终我们在主 Agent 的汇总工具里加了后处理：先尝试用正则提取 ``` 内的内容，如果失败再直接丢给一个“清理 JSON”的 LLM 调用。更稳健的方案是使用 function calling 让子代理直接调用返回结果的 tool，我们后来引入了 `finish` 强制结束并携带 JSON payload。

2. **上下文膨胀**
   每个子代理的 prompt 中都会自动注入主 Agent 的任务描述和当前对话的部分上下文。如果不加控制，并行 5 个子代理可能导致大量上下文冗余。解决方案是在创建 SubAgent 时设置 `share_context=False`，只传递必要参数（如目标 URL），让子代理完全无状态运行。

3. **子代理间的间接耦合**
   最初主 Agent 会把前一个子代理的结果告诉下一个子代理（比如拿到了 A 站价格，让 B 子代理根据 A 的基准做分析）。这破坏了并行性，重新陷入串行。我们调整架构，让主 Agent 在拿到所有并行的“原子结果”后，再启动一个专门的“分析子代理”去比对，保证第一阶段完全解耦。

4. **日志追踪混乱**
   多个子代理同时运行，OpenClaw 的日志会交错。我们为每个子代理设置了不同的 `name` 并写入结构化日志，主 Agent 在最终报告里附上每个子代理的运行摘要（耗时、状态），便于回溯。

## 可复用建议

- **遵循“主轻子重”模式**：主 Agent 仅负责决策和汇总，子代理做具体脏活累活。这样主 Agent 保持上下文干净，也更容易调试。
- **为子代理定义明确的输入/输出契约**：使用 JSON schema 或 pydantic 模型约束子代理的返回，避免自由文本。
- **并发数不要盲目放大**：考虑 API 配额和每个子代理的 token 消耗，一般 3-5 个并行是安全区。
- **灰度引入与熔断**：先用两个子代理跑通，再加入更多；为子代理加失败计数，如果某个子代理连续出错，主 Agent 应能快速决定跳过并继续。
- **利用 OpenClaw 的 SubAgent 生命周期回调**（如果框架支持），在子代理完成/失败时自动记录指标，这对后续优化非常重要。

## 总结

通过 OpenClaw 的 Subagent 编排，我们把原来的串行任务变成多个 AI 子代理并行执行，显著缩短总耗时，同时隔离了不同子任务的上下文，降低了单一 Agent 的认知负担。实践中最关键的并不是技术细节，而是设计清晰的边界：主 Agent 做什么、子代理做什么、他们如何沟通。一旦这个契约建立，工程实现就变得顺畅。

这种模式不止适用于爬虫，我们在“多文件代码审查”“多渠道客服路由”“多数据源对比”等场景都复制了类似架构。后续我们计划结合 OpenClaw 的 MCP 连接器，让子代理能动态获取工具，进一步降低配置成本。

希望这篇基于真实踩坑的实践记录能对你的 Agent 并行编排设计有所启发。欢迎在 OpenClaw-CN 社区交流更多 subagent 使用经验。

---

