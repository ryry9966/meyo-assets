---
title: OpenClaw 的 session 隔离：子 Agent 不污染主会话的工程化做法
feedId: 34616
source: 综合讨论
publishedAt: 2026-08-25
---

# OpenClaw 的 session 隔离：子 Agent 不污染主会话的工程化做法

## 背景

在 OpenClaw 的自动化流程中，主 Agent 经常把搜索、文件处理、代码生成等任务拆给子 Agent。如果子 Agent 直接在当前 session 内执行，中间过程会大量回灌主会话：工具调用参数、重试日志、MCP 返回片段、局部推理。这些内容不只是浪费 token，还会改变主 Agent 的决策上下文，让它把子任务里的噪声当成已知事实，或被过期工具结果带偏。

## 问题

常见的污染有三类：

1. **上下文污染**：子 Agent 的中间 tool call、错误重试、局部 prompt 全量进入主 session。
2. **指令污染**：子 Agent 默认继承父 system prompt，又叠加子任务提示，导致优先级冲突或角色越界。
3. **工具/状态污染**：子 Agent 复用同一个 MCP 连接或工作目录，读写副作用直接影响主任务环境。

## 做法/步骤

**第一步：建立独立子 session。**  
不要在主 session 内直接跑子任务。创建 child session 时显式声明隔离级别和返回策略，例如：

```yaml
subagent:
  isolation: strict
  inherit_system: false
  return_mode: final_only
  context_budget:
    max_tokens: 8000
    max_tool_rounds: 6
  tools:
    - mcp.search
    - mcp.read_file
```

**第二步：父 session 只接收 final answer 或结构化摘要，不接收 trace/message list。**  
代码层封装建议统一入口：

```python
child = parent.spawn(
    system="You are a file-search subagent. Return JSON only.",
    tools=["mcp.search", "mcp.read_file"],
    return_mode="final_only",
)
try:
    result = child.run(task)
    parent.add_tool_result({
        "role": "subagent",
        "summary": result.final_answer,   # 不回传 result.trace
        "child_session_id": child.id,
    })
finally:
    child.close()
```

**第三步：对 MCP 工具做边界。**  
子 session 应使用独立工具命名空间或只读凭证。需要写操作时，给子 Agent 指定 `sandbox_dir`，禁止覆盖父工作目录。工具输出在子 session 内消费，只把必要字段返回父级。

**第四步：子 session 结束后关闭或归档。**  
保留 `parent_session_id` 与 `child_session_id` 的关系，细节日志落入归档存储，而不是主上下文。

## 踩坑点

- **只设 `return_mode=final_only` 仍可能污染 memory**：如果全局 memory 默认 append 所有 session 消息，子 Agent 的中间过程还是会进入检索范围。需要把子 session 的 memory mode 设为 `none` 或 `parent_append`。
- **子 Agent 异常堆栈被当成正常输出**：如果 `child.run` 抛出异常而调用方没有捕获，错误堆栈可能被主 Agent 当作工具结果继续推理。必须捕获并返回结构化错误，如 `{"ok": false, "error": "..."}`。
- **并发子 Agent 共享状态**：多个子 session 同时读写同一文件或同一 MCP 连接会产生竞态。避免共享可变状态；MCP 连接池按子 session 隔离，或使用只读副本。
- **上下文预算过小导致 partial answer**：子 Agent 在未完成时被截断，可能返回看起来合理但不完整的内容。父级要做完整性校验，比如检查 JSON 字段是否齐全、是否包含结束标记；不完整则重试并扩大预算。
- **子 session 忘记关闭**：大量短任务会造成句柄和连接泄漏。统一用 context manager 或 `finally` 关闭。

## 可复用建议

1. 封装 `spawn_subagent` 统一入口：强制 `final_only`、超时、错误结构、预算，不让业务代码随意创建裸 session。
2. 子 Agent 输出用 JSON Schema 校验，禁止自由文本进入主上下文。
3. 父 session 只保留摘要 + `child_session_id`，需要细节再按 id 查归档。
4. MCP 工具按风险分级：只读工具默认开放，写工具需要显式 sandbox 或审批。
5. 回归测试关注 token 增长：连续跑 N 个子任务，主会话上下文增长应接近 O(1)，而不是 O(N)。
6. 记录父子 session 关系，便于追踪某次错误来自哪个子任务。

## 总结

OpenClaw 的 session 隔离不是“少复制几条消息”的问题，而是四个边界：**上下文边界、工具边界、状态边界、生命周期边界**。在创建子 session 时定义清楚，比事后清理主会话更可靠。父 Agent 应该像调用一个外部服务一样使用子 Agent：只关心输入、输出、错误码，不关心其内部过程。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/84aaf1361a6e13f0.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/d80de58a359d7bda.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/279f0c4d87fda6ed.png)

