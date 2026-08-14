---
title: OpenClaw 子 Agent 会话隔离：别让工具日志写回主上下文
feedId: 33086
source: 综合讨论
publishedAt: 2026-08-14
---

## 背景

在 OpenClaw 里用子 Agent 做检索、代码执行或浏览器自动化时，很容易出现一种“越跑越笨”的现象：主会话上下文快速膨胀，后续决策被无关工具输出、中间推理、重试日志干扰。常见表现是主 Agent 开始重复读文件、误把之前的错误堆栈当成事实、或者指令跟随明显变差。

这通常不是模型能力问题，而是 session 边界没控制好：子 Agent 的“过程产物”被带回了主会话。

## 问题：污染路径主要有三类

1. **工具结果直接 append 到主 session**  
   子 Agent 跑 `search`、`code_exec` 时，如果工具层默认写当前 session，中间输出会直接进入主 `session_id`，而不是留在子任务里。

2. **final_answer 被合并而不是替换**  
   子 Agent 返回时，OpenClaw 可能把 `final_answer` 追加到 `messages` 尾部。主会话既保留了你最初的问题，也保留了子 Agent 的完整推导过程，造成上下文重叠。

3. **MCP/插件通过共享 session_context 写入元数据**  
   一些 MCP 工具会把 `tool_metadata`、`raw_stdout`、`error_trace` 写进共享上下文，主会话里出现大量碎片字段，进一步干扰主 Agent。

## 做法/步骤

### 第一步：给子 Agent 独立 session，不要 clone 父 session

创建子 Agent 时使用独立 `session_id`，只通过 snapshot 传入必要上下文：目标、约束、允许的工具、需要的文档 ID。不要直接传完整的 `conversation.messages`。

```yaml
child_agent:
  session:
    isolated: true
    inherit_memory: false
    snapshot:
      include: [goal, constraints, allowed_tools]
      exclude: [messages, raw_tool_logs]
```

### 第二步：限制写回策略

将子 Agent 的写回策略设为只返回最终状态，禁止追加 trace：

```yaml
child_agent:
  session:
    writeback: final_state
    tool_log: none
    max_rounds: 8
```

`max_rounds` 建议一并设置，防止子 Agent 在异常路径上无限重试。

### 第三步：在工具层加过滤

不依赖 Agent 自觉。在 MCP 工具或插件中间层加一个 `filter_result`，只保留结构化字段：

```python
def filter_result(r):
    return {
        "status": r.get("status"),
        "summary": r.get("summary", ""),
        "artifact_id": r.get("artifact_id"),
        "tokens": r.get("tokens"),
        "last_error": r.get("last_error"),
    }
```

去掉 `raw_stdout`、`stack_trace`、`intermediate_steps`。主会话只需要结论、证据引用和成本。

### 第四步：设置 session TTL 与清理

子 Agent 完成后标记 `session.status=closed`，延时清理。子 session 的 `messages` 不要回灌父 session。如果中间产物有价值，落到 `artifacts/` 或对象存储，主会话只引用 `artifact_id`。

## 踩坑点

- **只设 `isolated: true` 不一定够**  
  有些工具走 `session_context.write()`，会绕过 Agent 的写回策略。需要在工具配置里显式指定 `target_session: child` 或 `read_only: true`。

- **完全禁用写回会丢失排障信息**  
  建议保留 `last_error` 和 `retry_count` 两个轻量字段，不要保留完整 trace。否则子 Agent 失败后，主会话不知道是否需要重试。

- **并行子 Agent 共享 namespace 会互相覆盖**  
  多个子 Agent 如果共用同一个 `namespace` 或 `workspace`，缓存、临时文件、状态变量会串。务必为每个子 Agent 分配独立 `namespace`。

- **不要用 `session.clone()` 做隔离**  
  克隆会把已经存在的污染一起复制过来，主会话里的噪声和错误记忆会继续扩散。隔离应该从创建时开始，而不是复制后修补。

## 可复用建议

- 主会话只收“结论 + 证据 + 成本”三类信息，例如：  
  `{"conclusion":"...","evidence":["..."],"tokens":1234}`

- 给子 Agent 加 `session_group` 标签，例如 `group=research`，便于批量清理和观察。

- 监控主会话的 `context_watermark`。当非结构化工具日志占比超过 30% 时，基本可以判定隔离失效。

- 把子 Agent 当成无状态执行单元：输入最小上下文，输出结构化结果，中间过程留在可回收的独立 session 里。

## 总结

Session 隔离不是简单加一个 `isolated` 开关，而是边界设计、写回策略、工具层过滤和生命周期清理的组合。主会话只保留决策所需的最小信息，子 Agent 的过程产物留在自己的 session 中，用完即弃。这样才能避免主会话被工具日志和中间推理持续污染。

---

