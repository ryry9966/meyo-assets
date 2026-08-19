---
title: Agent 的 subagent 编排：多个 AI 并行做事的实践
feedId: 33861
source: 综合讨论
publishedAt: 2026-08-19
---

# Agent 的 subagent 编排：多个 AI 并行做事的实践

## 背景

在 OpenClaw 里做自动化和 Agent 编排时，很容易遇到一种情况：单个 Agent 既要查资料、又要调工具、还要写代码和生成报告。任务一复杂，上下文迅速膨胀，串行执行时间拉长，工具权限也越来越难控制。尤其是接入了多个 MCP server 或插件后，一个 Agent 拿着所有工具的访问权，出错面很大。

解决思路是把一个大任务拆成若干个小任务，交给多个 subagent 并行执行，主 Agent 只负责调度、汇总和校验。这种模式在 OpenClaw 的 Agent/MCP/插件体系里落地性很强，但并行不是银弹，实际做下来有不少工程化细节要处理。

## 问题

以一个典型场景为例：自动分析一个 GitHub 仓库，输出竞品调研、代码质量审计和部署文档。如果只用一个 Agent 串行做，耗时可能是三部分之和，而且上下文里混杂大量代码、网页内容和工具输出，最后生成的结果容易跑偏。

主要问题有三个：

1. **串行慢**：多个无依赖任务无法同时推进。
2. **上下文混乱**：不同子任务的数据互相干扰，主 Agent 难以区分来源。
3. **权限过大**：一个 Agent 绑定所有 MCP 工具，误调用风险高。

## 做法 / 步骤

我在 OpenClaw 环境中按以下方式落地 subagent 并行编排，核心原则是“无状态 + 最小权限 + 结构化输出”。

### 1. 拆分无依赖子任务

先画出任务依赖关系。竞品调研、代码审计、文档生成三者之间没有强依赖，可以并行。审计内部如果依赖 clone 仓库，就作为该 subagent 自己的前置步骤，不提升到主流程。

### 2. 定义独立 subagent

每个 subagent 只做一件事，绑定专属工具集。比如：

- `research-agent`：绑定网页搜索和 GitHub API 的 MCP server。
- `audit-agent`：绑定文件读取、代码分析工具，只给仓库目录只读权限。
- `docs-agent`：绑定文档模板和发布工具，不接触代码仓库。

输出统一为 JSON，字段固定，例如：

```json
{
  "task_id": "audit-001",
  "status": "success",
  "summary": "发现 3 个中风险问题",
  "data": { ... },
  "error": null
}
```

### 3. 主 Agent 并行调度

主 Agent 只做编排，不直接执行具体子任务。使用并行执行器同时启动三个 subagent，等待全部返回或超时。伪代码思路：

```python
results = await asyncio.gather(
    dispatch(research_agent, research_payload),
    dispatch(audit_agent, audit_payload),
    dispatch(docs_agent, docs_payload),
    return_exceptions=True
)
```

这样单个 subagent 失败不会中断整个流程。

### 4. 汇总与校验

主 Agent 拿到三个 JSON 结果后，只读取 `summary` 和 `status` 字段做汇总，不把原始子任务输出全部灌入上下文。需要详细数据时，再根据 `task_id` 定向调取，避免上下文爆炸。

### 5. MCP 工具隔离

在 OpenClaw 中，MCP server 的连接配置可以按 subagent 隔离。实际做法是每个 subagent 配置独立的 MCP 白名单，比如 `audit-agent` 只挂文件系统 MCP，不挂浏览器或发布工具。这样即使 subagent 指令跑偏，也没有权限造成大范围破坏。

## 踩坑点

1. **上下文爆炸**：subagent 返回过长，主 Agent 汇总时超限。解决：强制每个 subagent 输出不超过 500 字的 `summary`，详细数据放 `data` 字段，需要时再二次查询。
2. **共享状态竞争**：多个 subagent 同时写同一个文件或数据库会冲突。解决：每个 subagent 使用独立工作目录，最终产物由主 Agent 统一落盘。
3. **并行度失控**：同时跑太多 subagent 会导致 API 限流或本地资源吃紧。建议控制并发数在 3~5 个，视任务类型调整。
4. **错误处理缺失**：必须用 `return_exceptions=True` 或类似机制捕获异常，否则一个子任务超时可能拖垮整个 gather。
5. **结果顺序不确定**：不要依赖 subagent 完成顺序，统一用 `task_id` 做映射。

## 可复用建议

- **先串行后并行**：先让三个 subagent 串行跑通，确认输出 schema 没问题，再切到并行，能省很多排障时间。
- **无状态设计**：subagent 不保存全局状态，每次调用都是独立输入输出，方便重试和横向扩展。
- **统一输出 schema**：所有 subagent 返回同一套 JSON 结构，主 Agent 汇总逻辑可以复用。
- **最小权限原则**：按任务分配 MCP 工具，宁可多建几个 subagent，也不要让一个 subagent 拥有所有权限。
- **加追踪 ID**：每次调度生成一个 `run_id`，贯穿所有 subagent 日志，方便事后排查。

## 总结

subagent 并行编排不是简单地把多个 Agent 同时跑起来，它更像一个微服务系统：任务拆分、无状态通信、结构化输出、错误隔离和权限控制缺一不可。在 OpenClaw 的 Agent/MCP/插件体系下，这套模式非常适合自动化流水线、批量数据处理和多源调研场景。控制好并行度和输出规范后，复杂任务的耗时能明显下降，同时可维护性反而比单 Agent 串行更好。

---

