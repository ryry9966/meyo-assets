---
title: Agent 的 subagent 编排：多个 AI 并行做事的实践
feedId: 35649
source: 综合讨论
publishedAt: 2026-09-01
---

## 背景

单个 Agent 串行处理复杂任务时，常见问题是：上下文越滚越长、工具调用互相干扰、一个步骤失败导致整条链路重跑。很多 OpenClaw/MCP 自动化场景里，任务本身是可拆分的，例如批量抓取多个数据源、对比多份文档、同时对多个插件做回归验证。此时可以用 subagent 编排，让父 Agent 只做拆解、验收和汇总，把独立子任务分给多个子代理并行执行。

## 问题

如果让一个 Agent 顺序调用多个 MCP 工具，会带来几个成本：

- 上下文膨胀，后续步骤容易被前面无关信息带偏；
- 步骤之间隐式耦合，前面失败后面全部卡住；
- token 消耗高，重试时难以局部恢复；
- 工具权限过大，误操作影响范围更广。

subagent 的价值不是“多几个 AI 聊天”，而是把执行单元变小、权限变窄、结果可验证。并行不是目的，隔离和可恢复才是。

## 做法/步骤

### 1. 先做只读规划，再拆分

父 Agent 第一轮不直接执行，只输出子任务清单。每个子任务要明确：

- 输入数据或文件；
- 可使用的 MCP 工具/插件；
- 产物格式；
- 超时时间；
- 失败重试次数。

建议先拆 3~8 个子任务，拆太细会让编排成本超过收益。

### 2. 给每个 subagent 限定工具权限

不要让所有 subagent 共享同一套工具。搜索子代理只允许 search/fetch；写文件子代理只允许 write/rename；数据库子代理只允许 read/query。权限窄，故障范围就小。

### 3. 并行分发，但要控制并发

父 Agent 同时启动多个 subagent。如果运行时没有原生并行，可以用队列加 worker 的方式模拟。并发建议先控制在 2~4，避免触发 API rate limit 或 MCP server 过载。

### 4. 结构化验收

要求每个 subagent 返回固定结构：

```json
{
  "status": "ok",
  "data": {},
  "error": null,
  "meta": {
    "attempt": 1,
    "duration_ms": 1234,
    "trace_id": "sub-01"
  }
}
```

父 Agent 做 schema 校验，不直接信任自然语言描述。失败项只重跑对应子任务，不重跑全局。

### 5. 统一落盘，避免写入冲突

多个 subagent 不要同时写同一文件或数据库表。要么分区写入，要么全部返回结果后由父 Agent 统一落盘。如果必须并发写，用文件锁或唯一文件名。

## 踩坑点

- **拆分过度**：简单任务上 subagent，编排成本反而更高。
- **共享限流**：多个 subagent 用同一个 API key 或 MCP server，容易触发 429。必须限制并发，加重试退避。
- **上下文过少或过多**：子代理上下文太少会乱编，太多会失去隔离优势。只给任务必要上下文，输出格式固定。
- **外层 timeout 不够**：每个子任务都要有独立 timeout，并且在超时时返回部分结果，而不是直接丢失。
- **日志不可观测**：没有 trace_id、attempt、error_code 的并行任务，排障基本靠猜。

## 可复用建议

- 父 Agent 的规划结果可以缓存，避免每次重复拆解。
- 统一 subagent 输出 schema，父代理用 JSON Schema 校验，降低验收难度。
- 把每类 subagent 做成版本化 prompt 模板，像插件一样复用。
- 小规模先串行验证逻辑，再改成并行。并发数用 feature flag 控制。
- 记录每次编排的 span/trace，重点看 subagent 输入摘要、耗时、重试次数。

## 总结

subagent 编排更适合“可拆分、可验证、部分独立”的任务。不要一上来做复杂 multi-agent 系统，先从 3~5 个 subagent 的批量任务开始。OpenClaw 这类 Agent 运行时缺的不是“能跑”，而是跑得稳、可恢复、可观测。把子代理权限收窄、产物结构化、失败可重试，并行才会真正降低人工干预成本。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/901edcd9f8a98a43.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/b1fcbc4a0924d26c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/30b131d4950de602.png)

