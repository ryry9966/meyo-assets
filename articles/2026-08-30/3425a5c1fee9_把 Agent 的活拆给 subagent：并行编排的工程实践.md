---
title: 把 Agent 的活拆给 subagent：并行编排的工程实践
feedId: 35430
source: 综合讨论
publishedAt: 2026-08-30
---

# 把 Agent 的活拆给 subagent：并行编排的工程实践

## 背景

单 Agent 处理复杂任务时，容易陷入长链路：先查资料、再读代码、再写报告，串行耗时且上下文不断膨胀。subagent 编排的思路是让主 Agent 只做调度和校验，把独立子任务分给多个 subagent 并行执行。

在 OpenClaw/Agent/MCP 场景里，subagent 通常不是另起一个聊天窗口，而是运行时创建隔离上下文的工作单元，共享同一套工具/插件权限，但输入输出独立。真正要解决的问题不是“能不能并行”，而是“并行之后怎么收口”。

## 问题

并行不是银弹。实践中容易遇到三类问题：

- 任务边界不清：subagent 之间互相等待，或重复干同一件事。
- 上下文污染：subagent 返回大段原始日志，主 Agent 合并时直接超限。
- 副作用冲突：多个 subagent 同时写同一文件、同一 MCP 资源，导致状态不一致。

## 做法/步骤

1. 先拆任务，再拆 Agent。把任务分成可独立验证的单元，例如「查三份文档的差异」「审一个模块的边界条件」「抽取日志中的错误码」。有依赖的步骤留在主 Agent 串行。

2. 给每个 subagent 一个窄输入。不要把所有历史都传进去，只给任务所需片段、工具白名单、输出 schema、最大步数。

3. 用结构化输出收口。例如：

```json
{
  "status": "ok",
  "summary": "一句话结论",
  "evidence": [{"source": "doc_a", "quote": "..."}],
  "uncertain": []
}
```

4. 并行执行并设置超时。主 Agent 用 run/subagent 原语一次抛出多个任务，等待结果时设置 timeout 和 max_tokens。

5. 主 Agent 只做合并和校验。检查 schema、去重、标记冲突，必要时对单个失败任务重试一次；仍失败就降级为「主 Agent 自己串行处理」或返回部分结果。

## 踩坑点

- 把有依赖的任务并行化：例如 subagent B 需要 subagent A 先跑的迁移结果，结果 B 拿到空数据或旧数据。
- 没限制返回内容：subagent 把工具原始输出、中间思考都吐回来，主 Agent 合并时上下文爆炸。
- 共享写权限过大：多个 subagent 并发写同一路径或同一数据库 key，出现覆盖。
- 拆分过细：一个简单查询拆成 6 个 subagent，调度和通信开销远超收益。
- 没有可观测性：失败后只能看到一堆并行调用，无法定位是哪个 subagent 超时、哪个工具误用。

## 可复用建议

- 给 subagent 固定模板：Role / Input / Output / Constraints / Stop conditions。每个 subagent 的输出必须是最终结果，不返回过程日志。
- 默认只读工具：MCP 工具里，查询、搜索、读取放开；写操作、发送、删除默认拒绝或要求主 Agent 确认。
- 用 correlation_id 串联日志。每个 subagent 的创建、工具调用、结束都带同一个 trace_id，方便回放。
- 先从 2-4 个 subagent 开始。验证任务拆分和合并逻辑稳定后，再增加并行度。
- 记录 token、耗时、重试次数、失败原因。这些指标比「看起来很快」更能指导优化。

## 总结

subagent 编排不是让多个 AI 无脑并行，而是把调度、执行、校验三层分开。主 Agent 保持轻量，subagent 保持窄任务和结构化输出，工具权限保持最小化。这样并行带来的提速才不会被上下文膨胀和状态冲突抵消。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/cbae0f92e247fbfe.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/f3d4bec74061bdbe.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/6fe2c4dce39d0e45.png)

