---
title: Agent 的 subagent 编排：多个 AI 并行做事的实践
feedId: 35505
source: 综合讨论
publishedAt: 2026-08-31
---

## 背景

单 Agent 在处理长链路任务时，容易陷入三种困境：上下文膨胀、工具调用串行阻塞、局部失败污染全局。比如让一个 Agent 同时做网页抓取、数据清洗、报告生成，它很可能在某个工具超时后反复重试，最后带着半成品结果继续跑，输出不可信。

在 OpenClaw / Agent / MCP 这类自动化场景里，一旦流程变长，主 Agent 的推理链会越来越重。最常见的表现是：越到后面越容易丢步骤、改错文件、重复调用同一个工具，或者把失败结果当成成功结果继续写。

## 问题

如果要让多个 AI 并行做事，真正难点不是“同时调几个模型”，而是：

- 怎么拆任务才不会互相依赖？
- 主 Agent 如何不成为瓶颈？
- 并行结果回来了怎么合并、校验、仲裁？
- 某个 subagent 失败后，如何避免拖垮整条链路？

## 做法 / 步骤

### 1. 定义 Worker 边界

每个 subagent 只负责一个可独立验证的任务。比如代码审计 worker、文档提取 worker、测试用例生成 worker。不要让 subagent 读全量文件并直接总结，应该由 supervisor 先切好切片再分发。

### 2. Supervisor 只做三件事

拆任务、派发、收口。Supervisor 不直接改文件、不生成最终长文。它维护一个任务队列，把任务描述、输入引用、schema 约束打包成结构化 JSON 后派发给 worker。

### 3. 通信走结构化结果

subagent 返回的必须是统一结构，例如：

```json
{
  "task_id": "task_012",
  "status": "ok",
  "data": {},
  "errors": []
}
```

不要让 subagent 返回自然语言“我觉得大概可以了”。结构化结果才能做校验和合并。

### 4. 并行执行用 worker pool

在 OpenClaw 这类环境里，可以把每个 subagent 封装成 MCP 工具或插件调用。并行时使用异步任务队列，不要在主 Agent 的同一个上下文里循环等待。主 Agent 只负责派发和收口，不参与具体执行。

### 5. 汇聚时先校验再拼装

对每个结果做 schema 校验、非空检查、字段完整性检查。失败的 worker 可以重试一次，超过次数就走降级路径，例如返回“该模块未能生成”，而不是继续编造内容。

## 踩坑点

- **上下文爆炸**：subagent 若被传入整份文档，很快就会把 token 吃光。正确做法是 supervisor 先做拆分，只传必要切片。
- **并发写冲突**：多个 worker 同时写同一个文件或数据库 key，会出现覆盖。必须让 worker 只写自己的独立路径，由 supervisor 最后合并。
- **失败重试放大**：设置最大重试次数，并给每次重试加随机退避，避免某个工具限流后所有 worker 一起重试。
- **提示词互相污染**：不要所有 worker 共用一个 system prompt。每个 worker 的 prompt 应只描述自己的任务和输出格式，避免它“顺手”做别人该做的事。
- **无限递归**：subagent 再调用 subagent 时，必须限制最大深度，否则会出现树状爆炸。

## 可复用建议

- 先在串行模式下跑通全部任务，再开启并行；并行只改变调度，不改变单个任务逻辑。
- 给每个 subagent 设置 token 上限、超时、最大重试次数。
- 所有派发记录带 task_id 和 parent_id，便于追踪是哪一层、哪个任务失败。
- 在 supervisor 层做结果去重和冲突仲裁，尤其是多个 worker 可能返回相似内容时。
- 把 subagent 封装成可复用的 MCP 工具或插件，输入输出 schema 固定，便于版本演进。

## 总结

Subagent 编排的核心不是“同时跑多个模型”，而是把复杂任务拆成可验证、可隔离、可并行的小任务，并用结构化协议收口。工程上的关注点应放在边界定义、失败隔离和结果校验上。这样做出来的并行 Agent 系统才稳定，不会因为一个 worker 的异常拖垮整条链路。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/738f407c867114bb.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/1caf108448e6fcfb.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/beb84e5724768a4f.png)

