---
title: OpenClaw 实战：子 Agent 不污染主会话的分级隔离方案
feedId: 32955
source: 综合讨论
publishedAt: 2026-08-13
---

## 背景

在 OpenClaw 里，主会话通常承载用户指令、关键中间决策、工具调用记录和偏好信息。一旦接入插件、MCP 工具或子 Agent 做搜索、代码生成、长任务拆解，子 Agent 的过程性输出很容易回流到主会话。结果就是上下文膨胀、推理被带偏，甚至出现系统提示被冲淡、敏感中间变量持久化的问题。

这个问题不是“记录太多”，而是“记录层级错了”：主会话需要的是决策上下文，不是执行流水账。

## 问题拆解

实际踩过的几类污染路径：

- 子 Agent 返回对象原样写入 `messages`，包含 `raw`、`tool_calls`、`observations`、`intermediate_steps` 等字段。
- 多个子任务并行时，共享同一 session namespace，导致返回内容交叉写入、顺序错乱。
- 子 Agent 继承主会话完整 history 和 system prompt，误以为自己是主 Agent，输出角色混乱。
- 只靠提示词要求“不要污染”，没有在编排层做强制裁剪。模型可能会遵守，但工程上不可靠。

## 做法/步骤

### 1. 把子 Agent 当无状态函数

为每个子 Agent 定义明确的输入/输出契约，只允许返回结构化结果。例如：

```json
{
  "status": "ok",
  "summary": "完成 3 个搜索，命中 2 个相关结果",
  "artifacts": ["file://tmp/result-01.md"],
  "next_actions": []
}
```

禁止把 `raw`、`logs`、`debug` 直接放进主会话。

### 2. 子 Agent 使用 ephemeral session

在 OpenClaw 中为子 Agent 创建独立 session，不继承主 history。核心配置思路：

```yaml
sub_agent:
  session_policy: ephemeral
  inherit_history: false
  inherit_system_prompt: false
  output_schema: subagent_result.json
  log_sink: subagent_traces
```

只把必要参数通过 `payload` 传入，不给子 Agent 看到主会话全貌。

### 3. 通过 MCP 工具层做摘要

如果子 Agent 是通过 MCP server 暴露的，优先在 server 侧维护自己的上下文。工具返回时只给摘要，不给过程。

```json
{
  "mcpServers": {
    "research": {
      "env": {
        "RETURN_MODE": "summary",
        "MAX_TOOL_OUTPUT": "800"
      }
    }
  }
}
```

这样主会话接触到的内容天然是压缩过的。

### 4. 编排层只回传 final_answer

在调用子 Agent 后加一个 reducer/cleaner，强制丢弃中间过程：

```python
def sanitize_subagent_result(raw):
    return {
        "status": raw.get("status"),
        "summary": raw.get("summary", "")[:1200],
        "artifacts": raw.get("artifacts", []),
    }
```

不要信任子 Agent 的“成功返回”会自己保持干净。必须在编排层显式裁剪。

### 5. 设置上下文预算与降级

给主会话配置硬限制，避免单次注入撑爆上下文：

```yaml
main_session:
  max_injected_tokens: 1200
  truncation_policy: drop_oldest
  child_result_mode: final_only
```

当子 Agent 返回超过阈值时，自动降级为“详见附件/日志链接”，而不是强行塞进主会话。

### 6. 分离 trace 与日志

每个子 Agent 分配独立 `session_id` / `trace_id`，日志进入单独存储，如 `subagent_traces`。主会话只保留一行引用，方便排障时回溯。

## 踩坑点

- **不要原样返回对象**：尤其不要保留 `tool_calls` 和 `observations`，这些是过程字段，不是结果字段。
- **并发时要隔离 session_id**：即使子 Agent 无状态，也不能共享同一个 session namespace，否则会出现写入竞争。
- **不要继承 system prompt**：子 Agent 继承主系统提示后，容易把主 Agent 的约束当成自己的任务目标，导致输出语义偏移。
- **不要只靠提示词**：提示词可以作为辅助，但裁剪必须落到编排层，否则生产环境一定会出现不可控情况。
- **raw trace 不要进主会话**：调试信息可以保留在独立日志里，但主会话只需要 final/summary/artifacts。

## 可复用建议

- 把子 Agent 封装成带 I/O schema 的 tool，主会话只能看到签名和摘要。
- 统一维护一个 `sanitize_subagent_result()` 后处理函数，所有子 Agent 走同一出口。
- 生产环境用 `final_only`，调试模式才开放 raw trace，避免默认污染。
- 监控三个指标：主会话注入 token 数、子 Agent 调用次数、主会话压缩率。
- 设置告警：当单次注入超过阈值时，自动切换为附件模式，而不是继续填充主会话。

## 总结

OpenClaw 的 session 隔离不是“不记录”，而是“分级记录”。主会话保留决策上下文和最终产出，过程数据留在独立 trace 中。把子 Agent 当无状态函数、强制结构化输出、在编排层做裁剪，比单纯依赖提示词可靠得多。这样主会话才能保持轻量、稳定，不容易被过程性内容带偏。

---

