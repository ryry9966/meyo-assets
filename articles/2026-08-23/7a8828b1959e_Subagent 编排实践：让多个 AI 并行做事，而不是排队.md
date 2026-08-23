---
title: Subagent 编排实践：让多个 AI 并行做事，而不是排队
feedId: 34360
source: 综合讨论
publishedAt: 2026-08-23
---

## 背景

不少自动化任务表面上是一长串工作，实际上由多个独立小任务组成：批量检查 20 个 MCP 工具的可用性、为多个插件生成变更说明、同时抓取多个数据源、分别审查几个模块的代码。如果让一个 Agent 顺序处理，不仅慢，还容易把不同任务的上下文搅在一起，一处失败就影响整条链路。

在 OpenClaw 里，我一开始也是用单 Agent 串行跑，后来改成主 Agent 调度多个 subagent 并行执行。这个模式不复杂，但真正落地时会踩不少工程坑。下面记录一次批量分析 12 个插件 README 的实践。

## 问题

Subagent 并行看起来简单：拆任务、并发跑、收结果。但实际难点在三点：

1. 拆得太碎，主 Agent 协调成本高；拆得太粗，并行度不够。
2. 上下文隔离虽然好，但 subagent 缺少必要的全局信息时容易跑偏。
3. 多个 subagent 的输出格式不稳定，汇聚结果时会拖垮主 Agent。

## 做法/步骤

**1. 主 Agent 只做调度，不直接干活**

给主 Agent 的 system prompt 写清楚：只允许调用 subagent，不直接访问业务工具。它的职责是任务拆分、路由、校验和合并。

**2. 每个 subagent 定义独立角色和最小上下文**

例如代码检查 subagent 只拿文件路径和检查规则，不拿整个仓库的聊天记录。公共指令放在主 Agent 或共享 skill 里，subagent 只保留最短角色定义。

**3. 统一返回 JSON**

要求每个 subagent 返回固定结构：

```json
{
  "status": "success",
  "summary": "...",
  "findings": [],
  "next_action": "..."
}
```

主 Agent 只读这个结构，不做二次解释。

**4. 控制并发数**

我习惯用队列控制并发，通常是 3-5。不要一上来就开 10 个。并发过高容易触发 MCP 服务限流或连接超时。

**5. 校验结果**

主 Agent 收到结果后检查 `status` 是否为 success、`summary` 非空、`findings` 是否为数组。不合规的直接丢弃并重试一次，重试仍失败就标记为 `failed`。

**6. 汇总输出**

主 Agent 按任务 ID 排序，把多个结果合并成一份报告。只做格式统一，不重新解释 subagent 的观点。

在批量分析 12 个插件 README 的任务中，主 Agent 生成 12 个 subagent 任务，每个 subagent 读取一个 README 和对应 manifest，返回 JSON。并行度设为 4，总耗时 3 分 40 秒。之前单 Agent 顺序处理要 11 分钟以上。

## 踩坑点

**MCP 工具并发限制**

多个 subagent 同时调用同一个 MCP server 时，很容易出现连接超时或 429。我遇到过三个 subagent 同时跑浏览器 MCP，结果两个拿不到页面。解决方法是给有状态的工具（浏览器、数据库连接）加队列，或者给每个 subagent 配置独立连接。

**子任务隐性依赖**

比如一个 subagent 写文件，另一个要读这个文件。如果并行启动，读的会拿到空文件。需要先画依赖图，无依赖的才能并行。我通常用 `depends_on` 字段标记，主 Agent 按拓扑排序执行。

**输出格式不稳定**

即使 prompt 里要求 JSON，模型有时会加解释文本或省略字段。一定要在汇聚前做字段校验。可以只做简单字段检查，不必引入额外库。

**Token 成本失控**

每个 subagent 都有独立上下文，重复的 system prompt 会放大 token 消耗。把公共指令外置，subagent 只保留角色定义和当前任务输入。

**失败不阻塞全局**

某个 subagent 失败不要阻塞全部。主 Agent 设置超时，默认 120 秒。失败结果标记为 `failed`，最后在报告里单独列出。

## 可复用建议

- 先问自己：这个任务能不能拆成无依赖或弱依赖的小块？不能拆就不要上 subagent。
- 用一个模板生成 subagent prompt，包含角色、输入、输出 schema、禁止事项。不要每次手写。
- 并发数从 3 开始，跑到稳定后再上调。
- 给 subagent 的上下文越小越好，只给必要字段。全局信息由主 Agent 在最终汇总时补上。
- 保留每次 subagent 调用的原始 JSON 和耗时，方便后续排查。
- 在 OpenClaw 中使用 MCP 插件时，把有状态工具放到专门的 worker 里，避免多个 subagent 抢占。

## 总结

Subagent 并行不是银弹，但适合 IO 密集、任务独立、结果可结构化的场景。把主 Agent 当作调度器，subagent 当作一次性 worker，严格约束输出格式并控制并发，就能在不牺牲稳定性的前提下拿到并行收益。这个模式比写复杂 pipeline 更轻，也更适合在 OpenClaw 里反复使用。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/c7e88b4b403a3d2e.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/f02337de4622df97.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/e6fc3ee44edd42d9.png)

