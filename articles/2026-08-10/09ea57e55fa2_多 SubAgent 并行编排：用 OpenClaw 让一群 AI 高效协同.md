---
title: 多 SubAgent 并行编排：用 OpenClaw 让一群 AI 高效协同
feedId: 32290
source: 综合讨论
publishedAt: 2026-08-10
---

## 背景：从串行到并行的实际驱动

在自动化报告生成、代码审查、多维度数据分析等场景中，单个 Agent 逐项处理任务的串行模式已经越来越吃力。例如，对同一份财务报告同时完成合规检查、风险识别、关键指标提取，串行意味着三倍延迟，且每个步骤的等待时间累加严重拖慢整体反馈速度。  
既然现代 LLM 的后端已经是海量并发的基础设施，我们完全可以利用多 Agent 并行来压缩端到端耗时。在 OpenClaw 框架中，SubAgent 与并行编排器的组合正好为这类需求提供了工程化支持，同时还能对接 MCP 工具，让每个子代理拥有自己的技能集。

## 问题：并行并非仅仅“多开几个线程”

表面上，把任务拆开同时跑几个 SubAgent 就能提速，但一落地就会碰壁：

- **API 限流**：同一 API Key 的并发请求数或每分钟请求数（RPM）有硬限制，并行过多直接触发 429。
- **输出不可控**：就算给出了 system prompt，不同子代理返回的格式千奇百怪，汇总时解析成本极高。
- **上下文污染**：为“节省 tokens”而让子代理共享长上下文，容易导致跨任务幻觉，例如风险评估子代理看到了本应只属于合规检查的数据。
- **成本失控**：并行跑多个高 token 消耗的模型会成倍放大费用，尤其在子代理无节制地输出冗长分析时。
- **结果冲突**：当不同子代理对同一份输入给出矛盾结论，主代理必须有一套清晰的仲裁或整合策略。

## 做法 / 步骤

下面是一次真实工程实践中落地的并行编排方案，基于 OpenClaw 的 `SubAgent` 与 `parallel` 执行器（文中出现的 API 接口已做简化，便于说明思路）。

### 1. 定义职责清晰的 SubAgent

每个子代理的任务必须单一、无交叉，并强制要求结构化输出。我们利用 OpenClaw 的 `response_model` 参数约束输出 Schema，同时绑定必要的 MCP 工具（如搜索、数据库查询）。

```python
from openclaw import SubAgent, Agent
from pydantic import BaseModel

class RiskItem(BaseModel):
    level: str
    description: str
    recommendation: str

risk_agent = SubAgent(
    name="RiskAssessor",
    system_prompt="仅基于提供的报告内容进行风险识别。返回 JSON 按 RiskItem 结构。",
    model="claude-3.5-sonnet",
    response_model=list[RiskItem],
    mcp_servers=["financial_db"]
)

compliance_agent = SubAgent(
    name="ComplianceChecker",
    system_prompt="检查合规性。返回合规项与违规项列表，JSON格式。",
    model="claude-3.5-sonnet",
    response_model=...
)
```

### 2. 任务拆分与并行调度

主 Agent 负责拆解文档，将同一份报告分发给多个 SubAgent，使用 `parallel` 同时启动。

```python
from openclaw import parallel, Agent

main = Agent(model="gpt-4o", system_prompt="...")
doc = load_report()

results = parallel({
    "risks": risk_agent.run(doc),
    "compliance": compliance_agent.run(doc),
    "metrics": metrics_agent.run(doc)
})
```

`parallel` 内部管理 asyncio 事件循环，并允许配置 `max_concurrency` 来控制同时发出请求的上限，避免触发 API 限流。

### 3. 结果聚合与冲突消解

拿到结构化数据后，主 Agent 负责整合。如果出现矛盾（例如风险评估认为无风险但合规检查发现违规），由主 Agent 通过二次查询或人工标记上升处理：

```python
merged = main.run(
    "根据以下子代理结果生成最终报告，如有冲突请指出：\n"
    f"风险：{results['risks']}\n合规：{results['compliance']}"
)
```

## 踩坑点与解决记录

- **429 风暴**：最初我们直接用无限并发，5 秒内触发限流。后来在并行调度前检查 API Provider 的 RPM/TPM，并用 `max_concurrency=5` 配合指数退避重试（自定义 `retry_with_backoff` 装饰器），问题解决。OpenClaw 的 `parallel` 支持传入 `rate_limit_rpm` 参数，内部使用令牌桶控制。
- **输出仍是自然语言**：即便在 system prompt 写“请返回 JSON”，模型偶尔还是会加前言。我们强制设置 `response_model` 底层启用 JSON mode 并配合 Pydantic 校验，校验失败自动重试一次，成功率大幅提高。
- **共享上下文幻觉**：曾经把整份 PDF 全文塞进每个子代理的上下文，导致风险评估引用了合规数据。改为由主代理做“片段剪裁”：为每个子代理只提供其分析所需的章节切片，成本下降且幻觉减少。
- **延迟不降反升**：并行 10 个轻量任务时，由于 LLM 服务端的排队，单个慢请求拖慢整体。我们设置了 `timeout_seconds=60`，并对关键任务使用快速模型（如 haiku 或 4o-mini），非关键任务用小模型，实现延迟与准确度的分级。
- **成本翻倍**：并行意味着并行消耗 tokens。我们为每个子代理设置了 `max_tokens` 限制，并且通过 `usage_tracker` 记录每次调用，生成每日成本报告，避免意外超支。

## 可复用建议

1. **先限流再优化**：做并行之前，先查出所用模型的 Rate Limit，并设置为并发上限的 80%，留出余量。
2. **结构化输出是必选项**：不要相信自然语言提示语能稳定返回 JSON。用框架的 `response_model` 或 function calling 强制结构，并编写校验重试逻辑。
3. **子代理隔离上下文**：不要让子代理“看到”不属于它任务范围的内容，由主代理做精简传递。
4. **分级模型策略**：关键路径用强模型，非关键路径用轻量模型，配合超时兜底，兼顾性能和成本。
5. **添加可观测性**：为每个子代理记录 request_id、耗时、token 用量和重试次数，方便事后调优。
6. **固化“并行编排模板”**：将上述任务拆分、并行调用、结果合并的模式封装为一个可复用的 `ParallelPipeline` 组件，后续新业务直接配置子代理列表即可。

## 总结

SubAgent 并行编排不是简单的多线程问题，而是一个涉及限流控制、输出稳定性、上下文管理和成本管控的系统工程。OpenClaw 提供的 SubAgent 抽象与并行执行器，让我们能够以较小的代码量把这些控制逻辑标准化。结合 MCP 工具扩展，每个子代理还能独立访问数据库、搜索引擎等外部能力，真正实现“一群 AI 同时做事”而不乱套。工程上，只要守住结构化输出、并发上限、隔离上下文这三条底线，并行就能变成可复用的提速手段，而不是一次性的 demo。

---

