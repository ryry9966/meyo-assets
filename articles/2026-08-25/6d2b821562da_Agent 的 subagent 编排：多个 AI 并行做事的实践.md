---
title: Agent 的 subagent 编排：多个 AI 并行做事的实践
feedId: 34681
source: 综合讨论
publishedAt: 2026-08-25
---

## 背景

单 Agent 处理复杂任务时，经常出现三类问题：一是串行执行慢，尤其是涉及网页抓取、数据库查询、文件写入等 I/O 密集操作；二是上下文越积越长，主 Agent 容易在长对话里丢失关键约束；三是工具权限混杂，一个 Agent 同时拥有搜索、写文件、发通知等能力，一旦误判影响面很大。

在 OpenClaw + MCP/插件体系里，一个更工程化的做法是 subagent 编排：主 Agent 只做任务拆解和结果汇总，把实际执行分派给多个只绑定最小工具集的子 Agent，并行完成。

## 问题

不是所有任务都适合并行。适合并行的场景通常满足：资源独立、无强依赖、主要耗时在外部工具调用。不适合的场景包括：后一步强依赖前一步结果、多个任务写入同一资源、子任务之间需要频繁交换中间状态。

实际落地时，最麻烦的往往不是“能不能并行”，而是“并行之后怎么不互相踩脚”。文件覆盖、数据库竞争、MCP server 会话冲突、结果格式不统一，这些都会让并行收益被排障成本吃掉。

## 做法/步骤

在 OpenClaw 场景下，我通常按四步走：

### 1. 拆分任务并定义子 Agent

每个 subagent 使用独立配置，只挂完成任务所需的最小工具集。例如做一次信息汇总：

- `search_subagent`：挂网页搜索、网页读取插件。
- `data_subagent`：挂数据库查询 MCP server。
- `report_subagent`：挂文件系统写入工具。

主 Agent 不直接执行这些工具，只负责派发。

### 2. 约定结构化返回协议

要求每个 subagent 返回固定 JSON：

```json
{
  "status": "ok",
  "summary": "...",
  "data": []
}
```

主 Agent 用这个协议做汇总，避免自然语言解析带来的歧义。`status` 可以是 `ok`、`failed`、`partial`，主 Agent 对 `failed` 要有兜底逻辑。

### 3. 并行触发

在 OpenClaw 中可以通过外部脚本或插件控制多个 subagent session。一个简单的调度伪代码：

```python
tasks = [
    {"id": "search", "agent": "search_subagent", "input": query},
    {"id": "data", "agent": "data_subagent", "input": db_params},
]

results = await asyncio.gather(*[run_subagent(t) for t in tasks])
merged = orchestrator.merge(results)
```

每个 subagent 拥有独立上下文，主 Agent 的完整历史不会自动传递给它们。派发时必须显式传入足够的输入参数。

### 4. 汇总与校验

主 Agent 收集结果后做冲突检测。例如如果 `report_subagent` 和另一个 subagent 都声称要写同一个文件，主 Agent 需要标记冲突并决定最终写入口。写操作最好收敛到单一出口，避免多 Agent 直接写文件系统。

## 踩坑点

### 1. 上下文隔离是特性也是坑

subagent 看不到主 Agent 上下文，这是为了保护上下文纯净，但也意味着关键信息必须显式传参。主 Agent 的“你以为它知道”很容易变成“它其实不知道”。

### 2. 并发写冲突

多个 subagent 同时写同一文件、同一数据库行或操作同一 MCP server，很容易互相覆盖。解决方式是按资源分片：一个 subagent 只处理 `data/` 目录，另一个只处理 `reports/`；或者写操作统一由主 Agent 执行。

### 3. 工具权限继承过大

如果 subagent 直接复用主 Agent 的全套工具，最小权限就失去意义。一个只需要读数据的 subagent 不应拥有写文件或发通知的能力。

### 4. 超时和部分失败

并行任务不能假设全部成功。每个 subagent 要设置最大步数和超时，主 Agent 必须容忍部分 `failed` 或超时返回，而不是直接中断整个流程。

### 5. 成本与观测

并行会放大 token 消耗，也会提高排障难度。每个 subagent 应记录 `task_id`、输入摘要、输出摘要、耗时和状态。没有这些日志，一旦出问题只能靠猜。

## 可复用建议

- 先串行跑通，再拆并行。不要一开始就上编排。
- 固定结构化返回 schema，减少主 Agent 的解析负担。
- 主 Agent 的 prompt 里明确写：“你是编排器，不要直接执行子任务的具体工具调用，只做任务分发、结果校验和冲突处理。”
- 给每个 subagent 独立命名空间或工作目录，减少资源竞争。
- 控制 subagent 最大步数和输出长度，防止死循环和 token 浪费。
- 先在开发环境用少量任务验证资源竞争点，再扩大并行度。

## 总结

subagent 编排的价值不在于“同时跑多个 AI”这个动作，而在于把复杂任务拆成边界清晰、可观测、可降级的执行单元。OpenClaw、MCP 和插件提供了工具层能力，但编排层需要自己控制权限、返回协议、失败路径和写出口。

工程上最实用的组合是：最小权限工具集 + 结构化返回 + 单写出口 + 可观测日志。做到这四点，并行才真正可控。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/379786af1eb62f7d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/86edd54ecf0f833f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/b625c74b03955a30.png)

