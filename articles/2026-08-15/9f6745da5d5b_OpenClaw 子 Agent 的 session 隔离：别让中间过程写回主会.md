---
title: OpenClaw 子 Agent 的 session 隔离：别让中间过程写回主会话
feedId: 33225
source: 综合讨论
publishedAt: 2026-08-15
---

## 背景

在 OpenClaw 里接浏览器自动化、MCP 工具链或多步检索任务时，很容易踩到一个隐蔽问题：子 Agent 是跑起来了，但主会话也被污染了。

典型表现是：主 Agent 只应该拿到“有没有找到目标页面、核心结论、失败原因”，实际却混入了大量 DOM 快照、重试日志、中间工具返回值。结果主上下文快速膨胀，后续规划被无关信息带偏，甚至把“失败重试中的某次局部成功”误当成最终结果。

这个问题不是 OpenClaw 独有的，但 OpenClaw 的子 Agent 机制里，session 边界需要显式设计，否则默认的回报链路会把你不想看到的东西带回来。

## 问题定位

主会话被污染，一般来自三个通道：

1. **消息回传**：子 Agent 内的中间 tool call、assistant 推理直接写回主 session。
2. **artifact 泄漏**：子 Agent 把中间产物 append 到主会话可读取的 artifact 区。
3. **MCP 输出未过滤**：MCP server 返回的 JSON 里夹带大字段，例如 `dom`、`cookies`、`network_events`。

很多开发者只在逻辑层做“任务完成没有”，没有在 session 层做“子 Agent 到底向主会话返回了什么”。等到上下文窗口告警，往往已经跑了十几轮。

## 做法与步骤

先说一个基本判断：子 Agent 的 session 隔离，关键不是“不让它写”，而是“只允许它按 schema 写回一份结构化结果”。

### 1. 启动子 Agent 时设置隔离策略

不要用默认的 spawn 方式。显式声明 `session_scope` 和 `return_mode`：

```yaml
subagent:
  name: browser_research
  session_scope: isolated
  return_mode: final_only
  result_serializer:
    type: compact
    fields: [title, url, core_facts, error_code, last_error]
  limits:
    max_tool_calls: 30
    max_context_tokens: 24000
    timeout_seconds: 300
  on_error: raise_compact
```

这里 `final_only` 很关键。它意味着子 Agent 内部的中间推理、工具调用、重试过程不会作为消息进入主 session，只有 final result 会被回传。

### 2. 过滤 MCP 工具输出

MCP 工具最容易漏大字段。可以在 OpenClaw 侧配置输出过滤器：

```yaml
mcp:
  tools:
    - name: browser_snapshot
      output_filter:
        exclude: [dom, cookies, network_events]
        max_bytes: 4096
```

如果 OpenClaw 侧不方便做字段级过滤，就在 MCP wrapper 里先截断，再返回给子 Agent。不要等子 Agent 拿到全量 DOM 后再决定扔掉——那已经占用子 Agent 上下文了。

### 3. 中间产物放子工作区，不回传主会话

子 Agent 经常需要写临时文件、截图、HTML 存档。建议直接在子 Agent 的独立工作区操作，final 阶段只回传 `artifact_id`、`sha256` 和简短描述。主 Agent 需要时再用 `artifact_id` 拉取，而不是一开始就全量塞进主 session。

### 4. 验证隔离效果

跑完后检查主 session 的消息列表。理想情况下，主会话中与子任务相关的内容应该只有：

- 主 Agent 调用子 Agent 的那条记录
- 子 Agent 返回的一条结构化 final result

如果主 session 里出现多条来自子任务的 tool 消息、中间 assistant 消息，说明隔离没生效，优先检查 `return_mode` 和 MCP 输出过滤。

## 踩坑点

**第一，以为 `isolated` 等于“完全隔离”。**  
实际上，子 Agent 的异常信息仍可能被拼回主 session。比如默认 `on_error` 策略会把完整 traceback 带回来。建议改成 `raise_compact` 或自定义 `on_error`，只回传 `error_code` 和 `last_error`，不要回传堆栈。

**第二，artifact 通道比消息通道更隐蔽。**  
很多团队只盯着消息列表，忽略了 artifact 区。子 Agent 可能把 5MB 的截图或完整 HTML 塞进 artifact，主 Agent 一旦读取，token 直接爆掉。处理方式很简单：**中间产物只存不读，final 只回传引用**。

**第三，并行子 Agent 共享命名空间导致污染。**  
如果没有给每个子 Agent 分配独立的 `session_id` 前缀或 `workspace`，并发执行时会出现变量覆盖、日志交叉写入的问题。建议给每个子任务生成唯一 `run_id`，并映射到独立工作区。

**第四，`final_only` 可能丢失关键路径。**  
完全只返回 final result 也有风险：主 Agent 拿到一个失败结果，却不知道失败在哪一步，无法决策重试还是换策略。所以 final result 里最好包含 `last_error`、`tool_trace_summary`、`steps_completed` 这类轻量诊断信息。

## 可复用建议

可以封装一个 `spawn_isolated` helper，统一处理子 Agent 的 session 策略：

- 默认 `session_scope: isolated`
- 默认 `return_mode: final_only`
- 统一返回 schema：`{status, summary, artifacts, error_code, last_error}`
- 强制设置 `max_tool_calls` 和 `timeout_seconds`
- 中间产物只写子工作区，不回传主 session

另外，建议给主会话加一个简单的监控：记录每轮主会话 token 增量。如果某轮增量异常大，基本可以判断是子 Agent 污染，这样能快速定位。

## 总结

OpenClaw 的子 Agent session 隔离，本质上是一个“边界设计”问题。

不是靠某个开关就能一劳永逸，而是要在三个层面都做约束：消息回传、artifact 引用、MCP 输出过滤。更重要的原则是：**子 Agent 应该像 API 一样工作，只返回结构化结果，不泄漏过程。**

把这条边界守住，子 Agent 才能真正成为可组合的自动化单元，而不是主会话里的噪声源。

---

