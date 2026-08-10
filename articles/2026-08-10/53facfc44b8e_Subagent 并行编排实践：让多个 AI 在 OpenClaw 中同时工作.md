---
title: Subagent 并行编排实践：让多个 AI 在 OpenClaw 中同时工作
feedId: 32389
source: 综合讨论
publishedAt: 2026-08-10
---

## 背景

在构建自动化 Agent 的过程中，单个 AI 串行执行任务很快就会碰到瓶颈：面对多个相互独立的子任务（例如同时审查 5 个代码仓库、并行抓取多源文档），顺序执行会让整体耗时线性累积。当任务规模一大，用户体验和实时性都变得不可接受。

OpenClaw 的 Agent 本身就支持工具调用、上下文管理以及 MCP 工具集成，但要让它“同时做多件事”，直接在一个对话里塞多条指令并不优雅，还容易把上下文搅乱。更工程化的做法是：**将主 Agent 作为编排者，动态创建多个 subagent 并行跑，最后汇总结果。** 这篇文章记录我在社区实践中的真实方案、踩过的坑，以及能直接复用的经验。

## 问题定义

假设一个典型场景：我们需要对 10 个 GitHub 仓库分别生成一份“代码质量简报”，简报包括检查 README 完整性、最近提交信息摘要、依赖是否过期。如果由一个 Agent 依次分析每个仓库，跑完可能需要十分钟；而如果能让 10 个 subagent 同时开工，就能把整体耗时压缩到最慢的那个仓库的时间附近。

要实现这个目标，需要解决几个核心问题：
- 如何安全地将任务拆解，并让每个 subagent 拿到**独立且清晰**的上下文？
- 如何管理并发，避免 token 消耗暴增或工具调用冲突？
- 如何收集结果，确保即使部分 subagent 失败也不影响整体？

## 做法与步骤

### 1. 定义标准化的 subagent 工厂

在实践中，我使用一个配置驱动的方法来创建 subagent，确保每个实例具有一致的 system prompt 和工具集：

```python
from openclaw import Agent

def create_subagent(task_id: str, tools: list) -> Agent:
    system_prompt = (
        f"You are subagent {task_id}. "
        "Complete the assigned task using the provided tools. "
        "Return a structured JSON object with keys: 'status', 'summary', 'data'. "
        "If an error occurs, set status to 'error' and include the reason."
    )
    return Agent(
        model="gpt-4o",          # 或社区常用的其他模型
        system=system_prompt,
        tools=tools,
        max_tokens=2000,         # 限制输出，防止膨胀
        stop_sequences=["[DONE]"]
    )
```

关键点：**每个 subagent 的输出必须结构化**，以便编排器统一解析。这里用 JSON 是一个低成本且可靠的选择。

### 2. 拆分任务并并行调度

使用 `asyncio` 管理并发，我一般会设置一个信号量来限制最大并行数，避免 API 速率限制或本地资源耗尽：

```python
import asyncio
from typing import Any

async def run_subagent(agent: Agent, task: str) -> dict:
    return await agent.run(task)

async def orchestrate(tasks: list[str], tools: list) -> list[dict]:
    sem = asyncio.Semaphore(5)  # 最多同时跑 5 个 subagent
    async def bounded_run(task: str, idx: int):
        async with sem:
            agent = create_subagent(f"task-{idx}", tools)
            return await run_subagent(agent, task)

    coros = [bounded_run(task, i) for i, task in enumerate(tasks)]
    results = await asyncio.gather(*coros, return_exceptions=True)
    # 处理异常，避免一个子任务失败导致整体中断
    processed = []
    for res in results:
        if isinstance(res, Exception):
            processed.append({"status": "error", "summary": str(res), "data": None})
        else:
            processed.append(res)
    return processed
```

这样，无论是 3 个还是 30 个同类任务，都能以可控的并发度一次铺开。

### 3. 结果聚合与后处理

编排器拿到所有 subagent 输出的结构化结果后，可以进行合并、排序，或根据 `status` 字段决定是否重试。我一般会再做一次轻量的 LLM 调用，将所有简报压缩成一份总报告，但这一步是可选的。

## 踩坑实录

1. **上下文膨胀**
   即使设定了 `max_tokens`，subagent 在工具调用过程中也可能把大量中间数据塞进上下文，导致最终请求超出模型上限。对策：在 system prompt 中明确要求“工具输出过长时只保留关键数据”，并考虑在工具侧做截断。

2. **工具调用冲突**
   当多个 subagent 同时访问同一个文件系统或数据库时，曾出现过写入覆盖、死锁等问题。解决方案：为每个 subagent 分配**隔离的工作目录**，或通过队列串行化写操作。对于只读资源（如 API 查询）则无此问题。

3. **超时与重试**
   API 偶尔会返回 5xx 或超时，某个 subagent 长时间卡住会拖慢整个 batch。务必在 `Agent.run` 上加超时控制，并在编排层对失败的子任务进行重试（最多 2 次）。

4. **幻觉发散**
   如果任务描述不够具体，subagent 可能自由发挥，输出的 JSON 格式不一致。经验是：在 system prompt 中固定输出 schema，并在运行后做一次结构校验，不通过的丢弃并要求重跑。

## 可复用建议

- **统一工厂**：永远不要手动拼接 system prompt，封装 `create_subagent()` 并复用。
- **限制并发**：通过信号量限制并行数，保护本地资源和 API quota。
- **强约束输出**：要求 subagent 返回固定 schema，并在编排器侧添加轻量解析器。
- **隔离资源**：为每个 subagent 提供独立的临时目录或命名空间。
- **可观测性**：为每个子任务记录耗时、token 用量和状态，便于后期调优。

## 总结

Subagent 并行编排是一种典型的“用正确性换时间”的工程手段：拆解得好，能成倍提升吞吐；拆解不当，反而会增加排错成本。在 OpenClaw 的 Agent 体系中，利用 `asyncio` + 信号量 + 结构化输出，可以在不增加太多复杂度的前提下，让多个 AI 同时干活。这种模式尤其适合审查、数据采集、批量生成等可并行化的自动化场景。

如果你已经在用 MCP 工具或自定义插件，这种编排方式同样适用，只需将工具集传入工厂函数即可。实际跑起来后，你会发现：**一个可靠的结果聚合器，往往比十个裸奔的 subagent 更有价值。**

---

