---
title: OpenClaw 多 subagent 并行编排：拆分、调度、限流与回收
feedId: 34801
source: 综合讨论
publishedAt: 2026-08-26
---

## 背景

在 OpenClaw 的 agent 实践中，单个 agent 串行处理多个独立任务时，通常会出现三类问题：上下文线性膨胀、整体耗时叠加、错误定位困难。比如同时抓取多个 API、批量执行插件命令、生成不同类型的文件。串行不是不能跑，而是每切换一次任务都要重复携带历史上下文，越往后越慢，token 成本也越高。

## 问题

如果只是简单“多开几个 agent”，会遇到更多工程问题：多个 subagent 同时写同一目录、工具权限过大导致误操作、返回结果格式不统一、API 限流、一个任务卡死拖住整批。真正需要的是一个轻量编排层：拆分、调度、回收、重试、追踪。

## 做法步骤

### 1. 按资源边界拆任务

不要按步骤拆，而按“谁能独立完成一个可验证结果”拆。比如数据获取 subagent 只负责拉取并清洗数据，代码生成 subagent 只负责根据接口定义产出文件，文档 subagent 只负责汇总已有结果。每个 subagent 只接收必要上下文，不携带全局聊天记录。

### 2. 定义输入输出契约

主 agent 生成任务清单，每个任务包含 `task_id`、`type`、`payload`、`timeout`。输出要求 JSON：

```json
{
  "task_id": "fetch_001",
  "status": "ok",
  "result": {},
  "error": null
}
```

主 agent 用 schema 校验，格式不对直接重试。

### 3. 调度与限流

OpenClaw 里可以用 subagent 调用或 MCP 暴露的任务执行器。主 agent 做并发调度，并发数建议控制在 2-4。如果多个 subagent 共用同一个 LLM provider 或同一个 MCP server，并发过高容易触发 429 或 server 超时。重任务用队列，而不是无脑 `gather`。

### 4. 收集与重试

主 agent 只做 merge，不逐字改写 subagent 结果。失败任务保留错误信息，重试一次；重试时降低并发，必要时换更小模型跑降级路径。

### 5. 日志追踪

每个 subagent 的 prompt 中加入“输出必须携带 task_id”。日志里记录 `task_id`、耗时、token、重试次数，便于判断是模型问题、工具问题还是调度问题。

## 踩坑点

- **上下文污染**：把主 agent 的全局上下文全量传给 subagent，会让 subagent 产生不必要的“判断”，并大幅增加成本。只给任务相关的文档和工具说明。
- **工具权限过大**：多个 subagent 如果共享可写工作区，会出现 A 生成的文件被 B 覆盖。建议每个 subagent 使用独立工作目录，或通过 MCP 资源服务做读写隔离。需要写共享文件时，由主 agent 单独串行写入。
- **结果格式不稳定**：要求 subagent 返回 JSON，但模型经常返回“好的，结果是...”或漏字段。不要只靠提示词保证，要加解析器和 schema 校验，失败自动重试。稳定后再上并行。
- **并发触发限流**：尤其多个 subagent 共用同一个 API key。建议做并发池，并对重试使用指数退避。
- **汇合点阻塞**：所有结果等待最慢任务。给每个任务设置 timeout，超时后返回降级结果或标记 partial，不要卡死整批。

## 可复用建议

- 任务队列模式比“主 agent 直接对话分派”更稳：主 agent 生成 `tasks.json`，subagent 消费 `task_id`，主 agent 再读取结果。中间状态落盘，避免调度中断后丢任务。
- 为每个 subagent 写固定 prompt 模板：角色、输入、输出 JSON schema、禁止事项、失败时返回错误对象而不是解释。
- MCP 资源最小化：只给 subagent 暴露完成任务需要的工具和资源。能用只读工具，就不要给写权限。
- 先串行跑通一个任务，再逐步开并发；不要一上来就并行。
- 记录每批任务的耗时和成本，找出最慢的 subagent 做拆分或优化。

## 总结

Subagent 并行不是“多开几个 agent”那么简单。对 OpenClaw 用户来说，真正有效的是把并行当作一个系统问题处理：明确任务边界、强制结构化输出、限制并发和权限、做好日志追踪。把这套基础打稳后，多个 AI 并行做事才会从“看起来很酷”变成可复用的工程能力。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/77f1691436159f43.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/70705d08e6255e00.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/3e9c9b313ab4be70.png)

