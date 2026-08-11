---
title: OpenClaw 中 Subagent 编排实践：如何让多个 AI 并行处理长链路自动化任务
feedId: 32608
source: 综合讨论
publishedAt: 2026-08-11
---

## 背景：为什么单个 Agent 不够用

在 OpenClaw 的日常自动化实践中，我们常会遇到“长链路、多来源”的任务。例如，一个“竞品分析”工作流可能同时需要：

- 抓取 3 个网站的更新内容
- 提取关键信息并分类
- 与上次报告做差异对比
- 生成图表和邮件

如果用一个 Agent 串行处理，每一步都依赖上一步，总耗时至少是各步骤之和。当单个步骤本身也依赖慢速工具（如浏览器渲染、大模型摘要），几十秒甚至分钟级的延迟会让自动化体验大打折扣。

更关键的是，串行模式无法利用 OpenClaw 的插件池。比如我们已经有“网页抓取插件”、“语义分析插件”、“图表生成插件”，单 Agent 需要反复切换上下文，还容易在前一步失败时阻塞整个流水线。

这就是引入 **subagent** 并实现**并行编排**的直接动机。

## 问题拆解：并行编排要解决的三个核心点

1. **任务拆分与分派**：主 Agent 如何将一个大任务切成可并行执行的子任务？
2. **子 Agent 间的通信与结果汇聚**：多个 subagent 同时运行，如何收集结果并做最终决策？
3. **容错与资源竞争**：并行必然带来工具调用冲突、超时、部分失败，如何让系统“崩而不溃”？

下面的实践基于 OpenClaw 的 Agent 框架（支持通过 `agent.run()` 创建子 agent，并可结合 MCP 工具）。我用一个“多渠道舆情日报”的自动化实例，展示具体做法。

## 实践步骤

### 1. 定义主 Agent 的职责与触发方式
主 Agent 只做三件事：接收指令 → 拆分任务 → 汇聚结果生成日报。它不直接调用外部工具，而是创建 subagent 分派工作。

在 OpenClaw 中定义主 Agent：

```python
main_agent = Agent(
    name="Orchestrator",
    instructions="你是任务编排器。...",
    tools=[]  # 主 Agent 不直接执行外部任务
)
```

触发方式可以是定时任务，也可以是通过 CLI 或 Webhook 传入关键词。

### 2. 拆分任务并创建并行 subagent
输入：“今天关注 AI 芯片的最新动态”。主 Agent 的主要逻辑是生成 subagent 的启动指令，然后收集结果。

关键代码片段：

```python
import asyncio
from openclaw import Agent

async def run_parallel_subagents(keywords):
    sources = ["网站A", "网站B", "研报源C"]
    tasks = []
    for src in sources:
        sub = Agent(
            name=f"Fetcher-{src}",
            instructions=f"你负责从{src}抓取和摘要关于'{keywords}'的信息。",
            tools=[web_scraper, summarizer]  # 注册好的插件
        )
        tasks.append(sub.run_async(f"搜索并总结最新的{keywords}信息"))
    results = await asyncio.gather(*tasks, return_exceptions=True)
    return results
```

这里通过 `asyncio.gather` 并行启动 3 个子 agent，每个子 agent 都配备相同的工具集。`return_exceptions=True` 是关键——任何子任务抛出的异常都会被捕获，不会中断整体。

### 3. 汇聚结果与二次加工
主 Agent 收到 `results` 列表后，过滤掉失败项，再调用一个“整合型 subagent”做最终报告生成：

```python
success_results = [r for r in results if not isinstance(r, Exception)]
summary_agent = Agent(
    name="Summarizer",
    instructions="将多来源信息整合成结构化日报。",
    tools=[chart_drawer, email_sender]
)
final = await summary_agent.run_async(str(success_results))
```

这样，抓取、整合、发送三个阶段实现了流水线并行和结果汇聚，总耗时约等于最慢的那个抓取 subagent 的时间。

## 实际遇到的坑

### 坑1：子 Agent 对共享工具的竞争
我使用了同一个 MCP 网页抓取工具，多个 subagent 并发访问同一个网站时，触发了目标网站的速率限制。解决方式是为每个 subagent 分配独立的 session 标识，并在工具层增加指数退避。在 OpenClaw 中，可以在工具实例化时注入不同的资源标识。

### 坑2：大模型上下文炸弹
有的抓取结果非常长，直接传入整合 subagent 会超出 token 限制或大幅增加成本。解决办法：抓取 subagent 先本地摘要（用轻量 summarizer），主 Agent 只传递摘要文本，而非原始页面内容。

### 坑3：超时设置不当导致假性失败
默认 subagent 运行超时是 60 秒，但浏览器渲染可能接近 80 秒。通过 `run_async(timeout=120)` 设定合理上限，并监控哪类任务容易超时，单独优化。

### 坑4：结果顺序与对应关系丢失
`asyncio.gather` 保持顺序，但失败项会插入。用 `(src, result)` 的元组包装，失败时保留源信息，以便日志追溯。

## 可复用建议

- **拆分粒度**：每个 subagent 做“一个站点 + 一个操作”。过细会增加编排开销，过粗则丧失并行优势。建议以外部数据源或独立计算任务为边界。
- **错误隔离**：所有 `run_async` 都要设 `return_exceptions=True`，主 Agent 必须能做降级处理（例如少一个来源仍生成简报）。
- **统一日志契约**：子 agent 返回结构使用 `{"source": "...", "content": "...", "error": null}` 的字典，方便主 Agent 统一解析。
- **资源限制**：使用信号量控制同时运行的 subagent 数量，防止 API 速率限制打满。
- **预热与复用**：如果 subagent 反复创建成本高，可设计 Agent 模板，用不同参数快速实例化，而不是每次都重新加载工具链。

## 总结

引入 subagent 并行编排后，“多渠道日报”总耗时从平均 110 秒下降到约 40 秒（取最慢来源的响应时间）。更重要的是，单个来源的失败不再阻塞整个报告生成，自动化流程的鲁棒性显著提升。

在 OpenClaw 的工程实践中，把每个 Agent 当作“可调度的执行单元”，通过主 Agent 的组织能力、异步并发和标准化结果格式，能够相对低成本地把串行流水线改造成弹性并行系统。这个思路还适用于多语言客户工单分类、代码质量检查多维度扫描等场景，值得在你的下一个自动化项目中尝试。

---

