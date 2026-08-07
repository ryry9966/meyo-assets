---
title: Subagent 编排实战：让多个 AI 并行处理复杂任务
feedId: 31973
source: 综合讨论
publishedAt: 2026-08-07
---

## 背景

在 OpenClaw 的自动化实践中，越来越多的工作流不再由单一 Agent 完成。当一个任务需要同时拉取多源数据、并行分析多个文件，或对一组候选方案独立评估时，线性执行会迅速变成瓶颈。受限于单次推理的上下文窗口和响应延迟，即使模型能力再强，也无法在合理的时间内完成高吞吐的并行处理。

常见的解法是引入 subagent（子智能体）编排：主 Agent 只负责任务划分、调度与结果汇总，实际的“脏活”交给一组功能单一、边界清晰的 subagent 并行执行。这个思路在工程上可行，但落地时状态管理、并发控制、错误恢复等问题会密集出现。本文记录一次真实的并行编排实践，不依赖特定云平台，全部基于 OpenClaw 内置的异步能力与 MCP 工具协议实现。

## 问题拆解

我们当时的场景是：为一批代码仓库生成安全摘要并提出修复建议，每个仓库有独立的文件树和规则集。目标要求在 2 分钟内处理完 20+ 个仓库，且结果必须满足统一的结构化输出，以便后续汇总报告。

显然，如果串行调用一个 Agent 逐个分析，光是等待 API 响应就会超时。更麻烦的是，不同仓库的分析难度差异大，简单的可能 5 秒完成，复杂的需要 40 秒，简单的阻塞会让后面的仓库空等。

我们决定采用 **主 Agent + 并行 subagent** 架构：
- 主 Agent 负责拉取仓库列表，切分为独立任务；
- 每个任务分配一个 subagent，subagent 只做“分析一个仓库 → 输出指定 JSON”这一件事；
- 利用 `asyncio` 控制并发，所有 subagent 同时启动，主 Agent 等待全部完成后合并结果。

## 做法与步骤

### 1. 定义 subagent 的接口与输出 Schema
subagent 必须是无状态的、可被并行安全调用的函数或工具。我们通过 MCP 暴露一个 `analyze_repo` 工具，接收 `repo_url`，严格按照 JSON Schema 返回结果。这样主 Agent 可以通过 `mcp_call_tool` 触发，也方便单元测试。

输出 Schema 示例：
```json
{
  "repo": "string",
  "summary": "string",
  "vulnerabilities": [
    {"severity": "high|medium|low", "description": "string", "file": "string"}
  ],
  "suggestion": "string"
}
```
注意：**强约束输出格式**。如果让 subagent 自由发挥，后续合并会变成灾难。我们使用了模型的结构化输出能力，结合 schema 校验，确保不合规的输出在 subagent 内部就被拦截并重试。

### 2. 并发调度器设计
主 Agent 拿到仓库列表后，动态创建 task 列表，使用 `asyncio.Semaphore` 限制最大并发数。这是踩坑后的经验——初期我们无限制并发 20 个任务，直接打爆了 API 速率限制，大量 429 错误导致不断重试，总耗时反而更长。

实现骨架（简化）：
```python
semaphore = asyncio.Semaphore(5)  # 根据 API 限制调整
async def run_one(repo):
    async with semaphore:
        return await mcp.call_tool("analyze_repo", {"repo_url": repo})
tasks = [run_one(repo) for repo in repos]
results = await asyncio.gather(*tasks, return_exceptions=True)
```
`return_exceptions=True` 是关键的容错开关：单个 subagent 失败不会拖垮整个批次，主 Agent 可以对异常做分类处理（重试、跳过、降级）。

### 3. 结果合并与校验
并行返回的结果可能是乱序的，我们在 task 定义时绑定了 `repo` 标识，确保结果能正确对齐。主 Agent 合并后先做全量 schema 校验，再进入后续汇总步骤。任何校验失败的项，根据失败原因决定是否用更保守的单线程重试。

## 踩坑点

1. **并发数设置的幻觉**  
   盲目设置高并发（例如 20）并不会线性提速。API 限流、本地资源竞争、模型推理服务的排队都会让收益递减。我们经过二分法测试，发现 5-8 个并发的端到端吞吐最高。最好根据实际 API 的 rate limit 和平均响应时间建模。

2. **subagent 返回格式漂移**  
   即便用了结构化输出约束，某些强推理场景下模型仍会返回多余的解释性文本，破坏 JSON 解析。对策是**双重保险**：在工具描述中明确“只返回合法 JSON，不要任何额外文字”；同时在解析前用正则提取首个 JSON 块，失败则触发重试。

3. **上下文污染与 token 浪费**  
   如果主 Agent 和 subagent 共享大量历史消息，并行时的上下文膨胀会非常惊人。我们的做法是 subagent 每次调用都从零开始构造 system prompt + user message，不携带全局历史。这还避免了并行任务间的非预期信息泄露。

4. **超时策略缺失**  
   初期没有设置 `asyncio.wait_for`，一个“卡住”的 subagent 会无限期占用信号量位置。后续我们给每个任务加了 60 秒超时，超时后自动取消，并记录日志用于事后分析。

## 可复用建议

- **封装 Orchestrator 类**：将并发控制、重试、超时、结果校验抽象成一个可复用的 `SubagentOrchestrator`，只需传入任务列表和执行函数。
- **信号量 + 指数退避重试**：对于 429 错误采用指数退避，避免重试风暴；其他逻辑错误（如 schema 非法）直接失败，不浪费重试配额。
- **输出 Schema 即文档**：把 subagent 的输出 schema 放在 MCP 工具描述里，主 Agent 可以据此做计划的动态调整，减少硬编码。
- **可观测性**：为每个 subagent 记录耗时、重试次数、最终状态，方便调优并发窗口和发现慢任务。
- **降级路径**：如果并行全部失败，至少保留一个串行兜底方案，保证核心链路可用。

## 总结

Subagent 并行编排并不是简单的“把任务拆开同时跑”，它需要细致的并发控制、严格的结构化契约和完备的错误处理。在 OpenClaw + MCP 的体系下，这套模式易于实现且可测试，能让自动化工作流在处理吞吐敏感任务时真正发挥多模型并行的优势。合理的抽象后，这个 Orchestrator 可以复用到代码审查、多源数据聚合、批量测试生成等多个场景，成为 Agent 工具链中稳定的一环。

---

