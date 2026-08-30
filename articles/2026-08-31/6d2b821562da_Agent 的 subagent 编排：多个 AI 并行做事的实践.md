---
title: Agent 的 subagent 编排：多个 AI 并行做事的实践
feedId: 35436
source: 综合讨论
publishedAt: 2026-08-31
---

## 背景

在 OpenClaw 这类 Agent/自动化链路里，单个主 Agent 串行处理多个任务时，常会遇到三个问题：响应慢、上下文膨胀、一个任务失败拖垮整条流程。比如同时调研多个开源项目、并行检查多个模块的 SQL、批量跑测试等场景，串行做法会非常耗时。

subagent 编排的思路是把大任务拆成边界清晰的子任务，主 Agent 只做分发、校验和汇总，多个 subagent 并行执行。它的价值不是“更多 AI 一起聊天”，而是任务分片、上下文隔离和更好的失败控制。

## 问题

并行不是银弹。真正的难点不在“能不能同时跑”，而在：

- 子任务是否真的无依赖，能不能并行；
- subagent 是否会乱调工具、产生大量无效输出；
- 并行结果如何汇聚、失败如何重试；
- 上下文和 token 成本是否反而增加。

## 做法/步骤

### 1. 先判断是否适合并行

适合并行的任务通常满足：输入独立、输出可结构化、无共享写状态、可幂等重试。比如“同时调研三个项目”“并行检查五个模块的 SQL 文件”。如果任务之间有强依赖，先画 DAG，不要强行并行。

### 2. 把 subagent 声明成最小能力单元

不要给每个 subagent 继承主 Agent 的全部工具。按角色裁剪工具、迭代次数和输出格式。一个示意配置：

```yaml
subagents:
  searcher:
    tools: [web_search, read_file]
    max_iterations: 8
    result_schema:
      type: object
      properties:
        summary: string
        sources: array
  reviewer:
    tools: [read_file, run_tests]
    max_iterations: 10
    result_schema:
      type: object
      properties:
        verdict: string
        issues: array
```

### 3. 主 Agent 只做分发和合并

主 Agent 的职责是拆解输入、调度 subagent、校验返回值、写最终结果。不要在主 Agent 里重复 subagent 的细节工作。可以通过 MCP 工具或本地 spawn 调用 subagent，优先使用结构化输入输出，避免自然语言来回转述。MCP 服务的好处是 subagent 可以独立重启、限流和升级，不会拖死主进程。

### 4. 并发控制与超时

同时跑 20 个 subagent 通常会触发限流或上下文失控。一般建议 3-6 个并发，按批处理。为每个 subagent 设置 `timeout`、`max_tokens` 和 `max_iterations`。

### 5. 结果只回传摘要

subagent 的最终返回应该是一个小体积 JSON 摘要，而不是完整日志或原文。需要原文时只返回文件路径或引用片段，由主 Agent 按需读取。

## 踩坑点

- **工具过多导致乱调**：subagent 拿到一堆用不上的工具，容易在错误路径上消耗迭代次数。按任务最小化工具权限。
- **共享文件冲突**：多个 subagent 同时写同一个文件或同一张表，很容易出现覆盖和脏数据。尽量每个 subagent 写独立目录，主 Agent 最后合并。
- **上下文爆炸**：subagent 返回大量无用内容，主 Agent 上下文快速耗尽。必须约定结果 schema 和长度上限。
- **重试没有退避**：失败就立即重试，遇到限流会更严重。加重试退避和最大重试次数。
- **可观测性差**：并行任务出错后只看到一堆报错，分不清哪个 subagent、哪个输入失败。给每次运行加 trace id。

## 可复用建议

- 把“并行分发-结果汇总”封装成一个固定流程，只替换 subagent 定义和输入模板。
- 小任务用 map-reduce 模式：主 Agent 分发，subagent 返回摘要，主 Agent 合并成最终报告。
- 使用 JSON schema 校验 subagent 输出，不合规就重试一次。
- 关键操作给 subagent 只读权限，写操作统一放在主 Agent 或专门的工作目录。
- 记录每个 subagent 的耗时、token 用量和失败原因，便于后续裁剪任务。

## 总结

subagent 编排的价值在于任务分片和上下文隔离，而不是单纯堆叠 AI 数量。工程上先做任务拆分，再做权限裁剪，最后做结果校验和并发控制。并行能提效，但只有把边界、超时、重试和日志做好，才真正可维护。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/a55513069e9f8508.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/d383b617f7e2880d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/b32cf4799834a78f.png)

