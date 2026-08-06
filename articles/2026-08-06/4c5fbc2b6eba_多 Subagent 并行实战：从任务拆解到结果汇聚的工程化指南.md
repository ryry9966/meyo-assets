---
title: 多 Subagent 并行实战：从任务拆解到结果汇聚的工程化指南
feedId: 31851
source: 综合讨论
publishedAt: 2026-08-06
---

## 背景：为什么需要 Subagent 并行
在 OpenClaw 这类 Agent 框架里，单个 Agent 串联调用工具已经能完成很多事。可一旦面对**多个相互独立的任务**（例如同时抓取 10 个网页、对 5 个 API 返回做分析、并行处理多份文档），顺序执行就会把总耗时拖到线性相加。工程上可行的解法是把任务拆成子任务，用多个 subagent **并行处理**，最后在主 Agent 中汇总。

但并行不是简单开几个线程——你需要解决上下文隔离、工具资源竞争、失败隔离和结果聚合的稳定性问题。本文基于实际搭建过的自动化流水线，给出一个可复现的工程化并行方案。

## 问题定义
假设主 Agent 收到请求：“分析这 6 篇论文摘要，提取各自的方法、结论，并排个优劣”。6 个分析动作完全独立，理想是 6 个 subagent 同时工作，耗时约等于单个分析最慢的那个。

直接开线程执行会遇到：

- **上下文污染**：多个 subagent 如果不加隔离地共享工具状态，容易出现变量覆盖。
- **资源冲突**：浏览器实例、API key、MCP 连接等资源可能没设计成可并发访问。
- **失败雪崩**：一个 subagent 崩溃或超时，如果没有熔断和汇聚策略，主流程会挂起或返回半成品。
- **结果格式不一致**：LLM 自由发挥，汇总时解析成本高。

## 工程化做法

### 1. 定义 Subagent 的标准化接口
给每个 subagent 指定**明确的输入输出 Schema**。在 OpenClaw 中，可以用 YAML 声明一个 subagent 的原型：

```yaml
subagent:
  name: paper_analyzer
  tools: [pdf_reader, web_search]
  input_schema: { type: object, properties: { paper_id: string } }
  output_schema: { type: object, properties: { method: string, conclusion: string, score: int } }
```

主 Agent 用这个 schema 生成任务列表，每个子任务就是一个 `paper_id`。

### 2. 主 Agent 的任务分解与 Plan 生成
利用主 Agent 的 LLM 先将复杂目标分解为 `PlanItem[]`，并严格按 output_schema 产出 JSON。例如：

```json
[
  {"task_id":"1","toolset":"paper_analyzer","input":{"paper_id":"paper_001"}},
  {"task_id":"2","toolset":"paper_analyzer","input":{"paper_id":"paper_002"}}
]
```
这样做的好处是**可验证、幂等**，重试时只需按 task_id 重放。

### 3. 并行调度引擎（核心实现）
在 Python 里用 `asyncio.Semaphore` 做并发控制，用 `asyncio.wait_for` 设超时。为每个 subagent 创建独立的上下文，避免工具状态共享。代码骨架如下：

```python
async def run_parallel(tasks: list[PlanItem]):
    sem = asyncio.Semaphore(5)          # 限制并发，防止触发 API 频率限制
    async def execute_one(item):
        async with sem:
            agent = SubAgent.from_yaml(item.toolset)
            return await asyncio.wait_for(
                agent.run(item.input), timeout=120
            )
    results = await asyncio.gather(*[execute_one(t) for t in tasks], return_exceptions=True)
    return process_results(results)
```

这里关键的隔离点：**每个 subagent 实例持有自己的工具会话和 LLM 调用上下文**，不和主 Agent 共享变量。如果使用 MCP 工具，需要为每个 subagent 分配独立的连接/会话 ID。

### 4. 结果汇聚与失败策略
`return_exceptions=True` 确保单个子任务失败不会中断全部。`process_results` 按成功/失败分类，对失败但可重试的（如超时）放入重试队列，对明确错误（如 PDF 不存在）标记并降级处理。汇聚时强制用 `output_schema` 校验，格式不合规的自动修复或丢弃。

## 踩坑记录

### 坑1：LLM 速率限制放大效应
并行后的瞬间请求数可能直接触发 API provider 的 RPM/TPM 限制。**解法**：Semaphore 控制并行度，并根据上游限制动态调整；同时在 subagent 调用链中加入随机退避重试。

### 坑2：浏览器实例等有状态资源冲突
假如 subagent 都用同一个 Playwright 浏览器上下文，一个 tab 崩溃会影响所有任务。**解法**：为每个 subagent 启动隔离的无头浏览器上下文（通过 `browser.new_context()`），用完立刻销毁。

### 坑3：错误传播与日志碎片化
并行乱序日志让排查变得困难。**解法**：所有 subagent 日志带 `task_id`，主 Agent 使用结构化日志（如 JSON）写入，最终可按 task_id 聚合分析。

### 坑4：结果丢失或重复
网络抖动导致部分结果没写入汇聚层。**解法**：结果先写入本地持久化队列（或 LiteDB），主 Agent 从队列消费，做到 at-least-once。

## 可复用建议
- **抽象并行层**：不要在每个任务里手写 asyncio.gather，封装一个 `ParallelDispatcher` 工具，主 Agent 通过工具调用即可。
- **幂等设计**：subagent 的输入必须包含唯一业务键，输出可校验。这样哪怕调度器重复执行，结果也安全。
- **工具资源池化**：把有状态工具（浏览器、数据库连接）做成资源池，便于 subagent 安全借还。
- **集成轻量级监控**：用 Prometheus 暴露子任务执行时长和成功率，方便发现并行瓶颈。

## 总结
Subagent 并行并不是银弹，但当你面对**天然可并行的独立子任务**时，一个经过工程加固的编排层能让端到端延迟从数十秒降到数秒。在 OpenClaw 的 Agent 工作流中，只要守住上下文隔离、严格 Schema 和失败隔离三道关，多个 AI 并行做事完全可以稳定落地。

---

