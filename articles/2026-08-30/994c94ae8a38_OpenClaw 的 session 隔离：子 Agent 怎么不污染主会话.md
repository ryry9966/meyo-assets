---
title: OpenClaw 的 session 隔离：子 Agent 怎么不污染主会话
feedId: 35362
source: 综合讨论
publishedAt: 2026-08-30
---

## 背景

在 OpenClaw 里，主 Agent 通常负责规划、拆解、调度，子 Agent 负责执行：跑 MCP 工具、搜索资料、改文件、调插件。问题在于，子 Agent 执行过程会产生大量中间结果：多轮 tool call、MCP 返回 JSON、重试日志、阅读摘要。如果这些内容直接回流到主 session，主 Agent 的上下文很快被挤满。

典型表现是：主 Agent 开始“忘事”、决策被细枝末节带偏、重试时被旧错误信息干扰，token 成本和长尾排障一起上升。

## 问题边界

这不是“子 Agent 能不能看主上下文”的问题，而是两个方向：

1. 读方向：子 Agent 是否要继承整个主 session 的 memory。
2. 写方向：子 Agent 的执行垃圾是否要回到主 session。

理想形态是：主 session 保留决策上下文，子 session 保留执行上下文；两者只通过一个受控接口交换输入输出。

## 做法/步骤

### 1. 独立 session 创建

不要用父级继承 memory。如果框架允许从当前 session fork，优先用“新 session + 显式传入 task brief”。例如：

```python
sub = runner.spawn(
    task=brief,
    session=Session(isolation="independent", share_memory=False),
    tools=mcp.subset(["search", "file_read", "code_exec"]),
    max_rounds=8,
    return_filter=["final_answer", "metrics"],
)
```

只传 brief 和必要 input payload，不要把主 session 的原始对话历史带进去。

### 2. 工具与 MCP 隔离

子 Agent 不要直接复用主会话的 MCP 客户端。最好给子 Agent 单独建连接，工具范围用白名单。MCP 返回体往往很大，子 Agent 内部要设置截断：

```yaml
max_tool_result_chars: 1200
truncate_mode: "head_tail"
```

大文件读取、日志 tail、搜索 snippet，都值得单独包装成“只读、截断、可重试”的工具。

### 3. 回传协议

子 Agent 结束时不要把 `messages` 或 `trace` 全量返回。让它返回结构化字段：

```json
{
  "status": "ok",
  "summary": "短摘要，说明结论和关键证据",
  "artifacts": ["file://..."],
  "errors": [],
  "metrics": {"rounds": 5, "tokens": 3200}
}
```

主 session 只追加这个结果，并在主记忆里记录 `trace_id`。完整执行日志落到外部文件或 OpenTelemetry，而不是塞进主 session。

### 4. 主 session 的写入控制

子 Agent 不能直接写主 session memory。若子 Agent 有写工具，应在插件 manifest 里关闭：

```yaml
write_main_context: false
write_parent_memory: false
```

子 Agent 只能通过 `return_filter` 回传受控结果。

## 踩坑点

- **`share_memory` 默认开启或继承**：新开子 Agent 时一定要显式关掉，否则子 Agent 会把主 session 的完整历史带进去，再把处理过的大段内容带回来。
- **只截断不索引**：子 Agent 内部截断后，排障信息会丢。所以回传结果里要有 `trace_id` 或 `run_id`，方便外部追溯。
- **MCP 工具未经包装**：某些 MCP 工具返回整个 sitemap、整页 HTML 或完整日志文件。子 Agent 拿到后可能继续全量引用，最终依然污染主 session。
- **错误栈全量回传**：子 Agent 异常时，容易把堆栈和失败工具结果一起返回。主 Agent 不需要这些，只回传 `status: error`、`error_code`、`retryable` 即可。
- **多级嵌套爆炸**：子 Agent 再 spawn 子 Agent，如果每层都带一点上下文，三层以后主 session 依旧被间接污染。建议限制 `max_depth=2`。

## 可复用建议

- 给每个子 Agent 固定 task brief 模板：角色、目标、可用工具、禁止写入、产出格式、最大轮次。
- 回传字段固定，至少包含 `status, summary, evidence, next_action`。
- 子 Agent 输出控制在 800 token 内；主 session 每次追加前做一次摘要过滤，避免摘要膨胀。
- 插件自动化场景，把子 Agent 封装成可重复调用的 skill，主 Agent 只看到“调用结果”，不看到内部执行细节。
- 排障时看外部 trace，不要试图在主 session 里翻子 Agent 历史。

## 总结

Session 隔离的关键是“回传面”和“可写面”都收窄。子 Agent 可以拥有完整执行上下文，但主会话只接收结构化结论和 trace 引用。这样既保留自动化能力，又避免主会话被执行垃圾拖垮。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/4e914f33dddc48e3.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/dbbcdf7cfc63eea9.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/b25e623081530bf7.png)

