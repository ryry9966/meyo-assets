---
title: Agent 的 subagent 编排：多个 AI 并行做事的工程实践
feedId: 35351
source: 综合讨论
publishedAt: 2026-08-30
---

## 背景

在 OpenClaw/Agent/MCP/插件/自动化实践里，单个 Agent 串行处理任务时，工具调用延迟和模型推理时间会不断累积。很多实际工作其实没有严格依赖关系：同时查询多个数据源、批量读取网页、并行跑多个插件、分别做不同文件的解析。Subagent 编排就是把一个主 Agent 当调度器，把可并行的子任务分给多个轻量 subagent 执行，再用结构化结果回收。

## 问题

直接“开多个 agent 一起跑”通常会翻车：

- 任务边界不清，多个 subagent 重复调用同一个工具/插件，产生副作用；
- 结果格式不统一，主 Agent 汇总时上下文爆炸；
- 某个 subagent 失败或超时，整个流程卡住；
- 并发过高触发 API rate limit 或 MCP server 过载；
- 调试困难，不知道哪个 subagent 在哪个环节改了状态。

## 做法

1. **先定义任务契约**：每个 subagent 的输入输出用结构化 JSON，不用自然语言。主 Agent 维护 `task_id` 到子任务的映射。
2. **按副作用拆分**：只读任务可以并行，例如搜索、读取、解析；写任务串行或加锁，例如更新数据库、发消息、修改文件。
3. **实现调度器**：主 Agent 生成 task list，由执行器池控制并发度，一般 3-5 个。每个 subagent 分配独立上下文、工具白名单、`max_steps` 和 `timeout`。
4. **收集结果**：subagent 返回 `{task_id, status, data, error}`，主 Agent 校验 schema，失败任务按策略重试或降级。
5. **监控与日志**：记录每个 subagent 的启动、结束、token 消耗、工具调用序列，避免“并行黑盒”。

一个信息采集流程可以这样拆：主 Agent 先创建 3 个搜索 subagent，分别查不同关键词；每个搜索 subagent 只读浏览器 MCP，不允许写入。搜索结果写入共享文件，清洗 subagent 再并行处理。最后主 Agent 合并入库。

## 踩坑点

- **切分过细**：subagent 本身有系统提示和上下文初始化成本。只对耗时或 I/O 密集且有独立边界的部分并行，拆成 10 个小任务可能比串行还慢。
- **共享状态竞争**：不要给多个 subagent 直接写同一个文件或数据库行。用不可变输入 + 显式输出，写入操作加 idempotency key。
- **无限重试**：失败重试必须设上限和退避，否则 token 费用会暴涨。要有 fallback 答案或部分成功返回。
- **上下文爆炸**：不要把所有 subagent 的原始输出都塞回主 Agent。让 subagent 写文件或 MCP resource，主 Agent 只读摘要和状态。
- **工具白名单缺失**：如果多个并行 subagent 都能调用“发送消息”插件，可能发出重复通知。按工具域拆分，或限制写工具只给单一 subagent。

## 可复用建议

- 从 2-3 个 subagent 开始，先跑通 contract 和汇总，再扩并发。
- 把 subagent 当纯函数：输入明确、输出结构化、副作用显式声明。
- 用共享工作区传递大结果，例如文件系统或 MCP resource，不要在消息里塞大 JSON。
- 给每个 subagent 设置预算上限和 kill switch。
- 读操作并行，写操作串行；能缓存的读操作不要让多个 subagent 重复调用。

## 总结

Subagent 编排的核心不是“让 AI 更聪明”，而是把工程约束前置：边界、并发控制、结构化输出、失败隔离。并行能显著降低端到端延迟，但只适合独立、I/O 密集、边界清晰的子任务。用主 Agent 调度、subagent 执行、结构化汇总的模式，在 OpenClaw 自动化里能做出稳定可复现的多 AI 协作流程，而不是演示性的一堆 agent 同时说话。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/4a9248b1f5916c01.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/65bc159dbb773761.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/63f4f4e4802b09a8.png)

