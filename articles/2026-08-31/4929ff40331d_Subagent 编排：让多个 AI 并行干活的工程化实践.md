---
title: Subagent 编排：让多个 AI 并行干活的工程化实践
feedId: 35557
source: 综合讨论
publishedAt: 2026-08-31
---

# Subagent 编排：让多个 AI 并行干活的工程化实践

在 OpenClaw 社区里，很多自动化任务从“单 Agent 跑通”走到“多 Agent 协作”时，会遇到同一个问题：任务量一大，单个 Agent 的上下文被塞满，执行速度变慢，而且一个步骤失败会让整条链路重跑。subagent 编排就是为了解决这类问题：主 Agent 不再亲自处理每个细节，而是把无依赖的子任务分发给多个子 Agent 并行执行，最后汇总结果。

## 问题：并行不是简单多开

直接把几个 Agent 同时启动并不难，难的是工程化。常见问题有三类：

1. 上下文污染：主 Agent 把完整对话历史复制给每个子 Agent，导致 token 成本高、响应慢。
2. 结果散乱：子 Agent 返回自由文本，主 Agent 很难稳定解析，合并时容易丢字段。
3. 错误不可见：某个子任务失败，但整个流程仍然继续，最终只得到一个不完整结果。

所以，subagent 编排的关键不是“更快地跑”，而是“拆得开、收得拢、看得见”。

## 做法：主 Agent 只做路由，子 Agent 只做单一任务

在 OpenClaw 中，如果版本支持原生 subagent，可以在配置里声明子代理；如果不支持，也可以把每个子任务包装成 MCP tool 或 plugin，由主 Agent 通过 tool call 调度。核心结构如下（示意）：

```yaml
# 示意配置：主 Agent 只做编排
orchestrator:
  role: router
  max_subagents: 4
  subagent_timeout: 120s
subagents:
  - id: crawl_docs
    tools: [web_fetch]
    output: json
  - id: summarize_issue
    tools: [read_issue]
    output: json
```

主 Agent 的 system prompt 只保留三件事：接收任务、拆解任务、合并结果。它不参与具体抓取、总结或写入操作。每个 subagent 拥有独立 system prompt、工具白名单和返回格式。比如抓取类 subagent 只允许 `web_fetch`，写入类 subagent 只允许 `db_write`，不允许交叉权限。

需要并行的任务要先满足一个条件：子任务之间没有数据依赖。例如“抓取 10 个文档并分别总结”可以并行；“先抓取 A 再根据 A 的结果抓取 B”只能串行。

返回统一用 JSON 包裹：

```json
{
  "status": "ok",
  "data": {},
  "error": null
}
```

主 Agent 拿到这些 JSON 后，只做合并与上报，不重新读子任务的原始日志。

## 踩坑点

1. **无界并发**：一次性拉起十几个 subagent，很容易触发上游 API 限流或本地资源耗尽。建议主 Agent 用固定并发数，例如 3-4，队列中多余任务排队。
2. **结果格式漂移**：让模型输出 JSON 时，必须给一个极小 schema，并且主 Agent 要做一层兼容解析，不能假设每个字段一定存在。
3. **共享写入冲突**：多个 subagent 同时写同一个文件或数据库表，会出现覆盖或锁等待。最好让每个 subagent 写独立临时文件，最后由主 Agent 合并。
4. **上下文过度复制**：只传给 subagent 完成当前任务所需的最小片段，不要把完整聊天记录、无关工具结果都塞进去。
5. **错误被吞**：每个 subagent 的返回里必须包含 `status` 和 `error`，主 Agent 要统计失败数量，并决定是否重试或降级。

## 可复用建议

- 任务粒度按“一次完成一件事、输出一个明确产物”来切。
- 给 subagent 限制工具白名单，不要默认继承主 Agent 的全部工具。
- 使用 trace id 标记每个子任务，日志里能看到哪条链路慢或失败。
- 先串行跑通一个子任务，再开放并行，否则错误会被并行放大。
- 记录 token 消耗、耗时和成功率，判断并行是否真的划算。小任务串行可能更省。

## 总结

subagent 编排不是让 AI 变得更快，而是让任务边界更清晰、失败更容易定位、结果更容易合并。OpenClaw 环境中，无论是原生 subagent 还是 MCP/plugin 包装，都可以用“主 Agent 路由 + 子 Agent 单任务 + JSON 返回”的模式落地。先把一个子任务跑稳，再逐步放开并发，比一开始就上复杂编排可靠得多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/c9c23b6d7d447a35.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/6c4ea05134b073a6.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/b54351e1254d39b7.png)

