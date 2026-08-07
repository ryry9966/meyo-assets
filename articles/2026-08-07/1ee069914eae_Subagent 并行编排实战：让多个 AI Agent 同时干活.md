---
title: Subagent 并行编排实战：让多个 AI Agent 同时干活
feedId: 32005
source: 综合讨论
publishedAt: 2026-08-07
---

## 背景

在 OpenClaw 的自动化流程里，单个 Agent 按顺序完成多步操作是常态：先查 A 表，再调 B 接口，最后汇总成报告。这在小任务上完全够用，但一旦任务可以明显拆分成几个独立部分，串行执行就开始拖慢整体耗时。比如同时从三个数据源拉取信息、并行处理一批文档的摘要，或者对同一输入跑多套分析模型再综合打分。

OpenClaw 本身就支持通过插件/MCP 工具调用实现协作，但早期实践中大家往往还是在主 Agent 里按顺序调用子功能，没有把并行潜力挖出来。subagent 编排——主 Agent 生成多个子 Agent 并行干活，再把结果收拢——是一种自然的加速手段，但做起来远比想象的琐碎：上下文隔离、并发控制、结果可靠性，每一个都是坑。

这篇帖子记录我在一个线上巡检流水线上，用 subagent 并行化改造的完整过程，希望对同样在用 OpenClaw 搭建自动化链路的朋友有点用。

## 问题定义

场景：一个巡检任务需要对 4 个独立系统做健康检查（A 库查询、B 服务 HTTP 健康探针、C MQ 队列深度、D 日志最新错误数），然后把结果拼成一份 Markdown 告警摘要。原来串行的流程耗时约 18–24 秒，因为外网请求和数据查询各自等待，单 Agent 上下文也越来越长。

改造目标：保持输出结构一致的前提下，将四个检查项并行化到一个主 Agent 调度中，整体耗时控制在 6 秒内（受限于最慢的独立检查），同时保证任何一个子任务失败不影响整体，结果能标记状态。

## 做法与步骤

### 1. 将子任务定义为独立的 subagent

在 OpenClaw 里，每个 subagent 本质是一个带有 system_prompt + 工具集的 mini agent。对于四个检查项，我分别注册了四个 subagent：

```python
# checks/subagents.py
subagent_specs = [
    {
        "name": "db_check",
        "description": "检查 A 数据库连接与慢查询",
        "tools": ["mysql_query_tool"],
        "prompt": "你是一个数据库巡检员，只返回 JSON 状态信息..."
    },
    {
        "name": "http_check",
        "description": "探活 B 服务接口",
        "tools": ["http_tool"],
        "prompt": "..."
    },
    # 类似为 mq_check / log_check 定义
]
```

主 Agent 通过 `use_subagent` 工具去调度它们。特别重要的是 **每个 subagent 的 prompt 里明确要求只返回结构化 JSON**，比如 `{"ok": true, "latency_ms": 23, "detail": "..."}`，这样主 Agent 合并结果时不需要再次解析自然语言。

### 2. 主 Agent 通过 parallel_tool 并行触发

主 Agent 不能只是循环调用 subagent，因为那样还是串行。我写了一个聚合工具 `parallel_checks`，内部用 `asyncio.gather` 同时触发四个 subagent 的运行：

```python
# tools/parallel.py
import asyncio
from openclaw import get_subagent_runner

async def run_parallel_checks(agent_ctx):
    tasks = []
    for spec in ["db_check", "http_check", "mq_check", "log_check"]:
        runner = get_subagent_runner(spec, parent_ctx=agent_ctx)
        tasks.append(runner.run("请执行检查并返回 JSON"))
    results = await asyncio.gather(*tasks, return_exceptions=True)
    # 处理异常封装
    return serialize_results(results)
```

主 Agent 的 system_prompt 只需要一句话：“当你需要全量巡检时，调用 `parallel_checks` 工具，并直接使用它返回的结构化结果生成报告。”

这样做的好处是主 Agent 上下文保持极简，它只看到已并行的结果摘要，而不会陷入每个子任务的冗长思考链。

### 3. 结果合并与容错

`asyncio.gather(return_exceptions=True)` 保证单个子任务挂了（比如 HTTP 超时）不会拖垮整体。我在 `serialize_results` 里把异常包装成统一的失败结构：

```json
{"source": "http_check", "ok": false, "error": "Timeout(5s)", "detail": null}
```

主 Agent 拿到的是一个结果列表，直接交给 LLM 生成最终报告。由于格式固定，模型能准确区分成功/失败项，并在摘要里标红失败部分。

## 踩坑点

### 并发与速率限制

一开始四个 subagent 同时抢同一个 MCP server 的 token，触发了后端服务的并发限流，导致部分请求直接被拒。解决方法是在 subagent 定义里增加 `max_concurrency` 参数，并为 MCP 工具加了一层客户端的 `asyncio.Semaphore(2)` 控制，避免被打。

### token 消耗爆炸

并行 subagent 各自的思考和工具调用会大量消耗 token。我的应对：
- 每个 subagent 使用尽可能低温度的模型（只要任务允许），不做多余推理。
- 限定 max_tokens，并强制要求仅输出 JSON。
- 关闭主 Agent 的“重述子结果”步骤，直接使用原始 JSON 生成 Markdown，不展开。

### 上下文隔离不够

OpenClaw 的 subagent 默认会继承部分父上下文，但并行场景下发现子 Agent 偶尔会看到其他子 Agent 的对话碎片（来自工具描述缓存），导致混淆。解决方法是在创建 runner 时传入干净的 `extra_context=""`，并确认 `parent_ctx` 仅共享必要的凭据/session id。

### 超时策略

如果某个子任务特别慢（比如数据库死锁），会拖累整体。我给每个子任务套了 `asyncio.wait_for(..., timeout=15)`，并在结果里标记“超时取消”。主 Agent 会提示该项未完成。

## 可复用建议

1. **子任务输出标准化**：强制 JSON 返回，并伴随 schema 校验（可用 Pydantic 包裹工具输出），能极大降低主 Agent 解析错误。
2. **封装通用并行工具**：不要在每个流程里重复写 gather 逻辑，抽象一个 `fan_out` 工具，入参是一组 agent_name + prompt，返回结构化列表。
3. **监控每一路**：为每个 subagent 记录单独的耗时和 token 用量，方便定位慢任务和高消耗节点。OpenClaw 的回调钩子可以轻易接入。
4. **降级串行方案**：当某些 subagent 必须顺序执行时（比如 A 的输出是 B 的输入），不要强行并行。可以设计一个 DAG 编排器，按依赖分组并行，用 `asyncio.gather` 分批。
5. **错误处理模板**：为子任务失败定义统一错误结构，并让主 Agent 的 prompt 包含“如果某检查返回 error，请使用 ⚠️ 标记，不要臆测原因”。这能避免幻觉补全。

## 总结

Subagent 并行编排在 OpenClaw 体系里并不复杂，真正花时间的是让每个子 Agent 输出可靠、控制并发、优雅处理失败。经过这次改造，巡检流水线耗时从 20 秒左右降到 4.2 秒（最慢的外部 HTTP 服务）。更重要的是，主 Agent 的维护成本大幅下降，增加新的检查项只需要注册一个新的 subagent，其余并行框架保持不变。

如果你的自动化流程里有很多可以分家的步骤，不妨试试以 `fan_out` 的方式重构。工程上只要管好结构化输出和超时，这套模式就能复用大半。

---

