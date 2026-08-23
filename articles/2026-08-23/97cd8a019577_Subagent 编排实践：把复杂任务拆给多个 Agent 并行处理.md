---
title: Subagent 编排实践：把复杂任务拆给多个 Agent 并行处理
feedId: 34304
source: 综合讨论
publishedAt: 2026-08-23
---

# Subagent 编排实践：把复杂任务拆给多个 Agent 并行处理

## 背景

在 OpenClaw 里跑自动化，单 Agent 很容易出现三类问题：上下文被中间结果灌满；长链路串行任务耗时长；一个工具失败拖垮整个流程。subagent 编排的思路，是把一个复杂任务拆成多个边界清晰的子任务，交给多个 worker agent 并行执行，最后由 orchestrator 统一汇总。

我遇到的实际场景是插件发布前检查：需要同时做代码规范检查、changelog 生成、基础测试和文档索引更新。如果让一个 agent 顺序做，它会在不同工具间反复切上下文，偶尔把测试输出当成文档内容写进去。更麻烦的是，任何一步失败都要整体重跑。

## 问题

subagent 编排不是简单多开几个对话，而是要显式定义：谁负责拆任务、谁执行、共享什么状态、如何合并结果。核心问题是：如何让多个 AI 并行做事，同时避免上下文污染、工具冲突和错误静默。

## 做法/步骤

### 1. 先串行跑通最小闭环

不要一上来就并行。先把任务拆成可独立执行的步骤，用单 agent 串行验证每步的输入输出。例如“代码规范检查”只接收 repo path，返回 JSON 结果。确认这样能跑通后再进入编排。

### 2. 定义 orchestrator 与 worker

在 OpenClaw 中配置一个 orchestrator agent 和多个 worker agent。orchestrator 只做三件事：拆解任务、调度 worker、校验合并结果。worker 保持单一职责，例如：

- lint-worker：只跑规范检查，输出结构化报告
- changelog-worker：只根据 commit 生成 changelog
- test-worker：只运行测试并返回通过/失败
- doc-worker：只更新文档索引

每个 worker 的 system prompt 明确“你只做 X，不接受其他指令”，减少越权和误操作。

### 3. 用 MCP 共享必要上下文，而不是全量复制

worker 之间不要互相看完整对话记录。把需要共享的状态放进 MCP server 或共享存储（如临时文件、Redis），比如 repo path、commit range、任务 ID。orchestrator 只把最小上下文分发给对应 worker，避免 token 浪费和内容串味。

### 4. 并行执行与超时控制

通过 OpenClaw 的任务队列或插件触发并行调用。每个 subagent 设置 max steps 和 timeout，比如 lint 5 分钟、test 10 分钟。不要让 worker 无限重试，失败就返回错误结构，由 orchestrator 决定重试或降级。

### 5. 结果校验与合并

所有 worker 返回后，orchestrator 用一个独立的“合并提示词”做最终汇总。要求 worker 输出 JSON 或 Markdown 固定结构，orchestrator 只读取关键字段，不重新推理。例如：

```json
{"status":"pass","issues":[],"artifacts":["changelog.md"]}
```

合并时如果某个 worker 返回失败，orchestrator 可以只重跑该 worker，不影响其他结果。

## 踩坑点

- **上下文隔离不够**：如果让所有 worker 共享同一段长日志，很快 token 就爆了，而且容易互相污染。只传结构化摘要。
- **并行依赖没理清**：有些步骤看似独立，实际有文件写入冲突。比如 doc-worker 和 changelog-worker 都可能改 README。需要提前划分输出目录。
- **错误处理缺失**：worker 失败时如果 orchestrator 直接吞掉错误，最终结果会静默丢步骤。必须让 worker 返回明确错误码。
- **工具冲突**：多个 subagent 同时调用同一个 MCP 工具（如 git commit）会竞争。对写操作加锁或让 orchestrator 统一提交。
- **token 消耗失控**：并行 agent 数量一多，总 token 成倍增长。限制 worker 数量（建议 3-5 个），单个 worker 上下文不超过 8k。

## 可复用建议

- 先串行验证，再并行优化。
- worker 只做一件事，输出结构化 JSON。
- 使用 MCP 作为共享状态层，不要复制大段上下文。
- 设置超时、最大步数、明确失败返回。
- orchestrator 的合并逻辑要简单，不要让它重新推理所有细节。
- 保留完整任务日志，方便回溯每个 subagent 的输入输出。

## 总结

subagent 编排的价值不是“看起来更高级”，而是把复杂任务切成多个可复用、可观测、可独立重试的小单元。在 OpenClaw 里落地时，优先考虑 worker 的边界和共享状态，而不是追求并行数量。把串行闭环跑通后，再逐步增加并行度，会少踩很多坑。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/07f10ce1ff02c101.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/a2dab3d255f706b9.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/e2abfeadeb993478.png)

