---
title: OpenClaw 的 session 隔离：子 Agent 怎么不污染主会话
feedId: 33232
source: 综合讨论
publishedAt: 2026-08-15
---

## 背景

在 OpenClaw 里，主会话承载用户上下文、历史消息、工具调用结果。主 Agent 调用子 Agent 处理子任务时，如果没有隔离，子 Agent 的中间推理、工具日志、MCP 返回、重试记录都可能写回主会话。结果就是上下文膨胀、关键信息被淹没、记忆层错乱。有人把子 Agent 当普通工具直接塞进主流程，很快主会话就超过 token 预算，或者用户下一次对话被残留的中间状态带偏。

## 问题

典型表现是：

- 主 Agent 调用子 Agent 后，主 session 多出几十条非用户可见的 step 日志；
- 子 Agent 的 system prompt 或工具描述泄漏进主会话；
- 子 Agent 因 loop/重试产生重复消息；
- 错误堆栈被当成正常上下文；
- 最终主 Agent 回答质量下降，甚至出现“记忆污染”，把子任务的临时变量当长期记忆。

本质不是“子 Agent 不能写主会话”，而是**子 Agent 的过程和结论没有做分离**。

## 做法/步骤

### 1. 为子 Agent 建立独立 session

创建子 Agent 时，不要让它复用当前 session_id。指定新的 session_id，通过 `metadata.parent_session_id` 关联主 session。关键是不要继承主会话历史。

示例配置片段：

```yaml
subagent:
  session:
    create: true
    parent_session_id: "${main.session_id}"
    inherit_context: false
```

`inherit_context: false` 是核心，避免子 Agent 把主会话历史全量拷贝过去。

### 2. 只回传结构化结果

子 Agent 完成后，不要把整个子 session transcript 写回主会话。让子 Agent 返回一个结构化 JSON，包含 `status`、`summary`、`artifacts`、`next_actions`。主 Agent 只用这个 JSON 更新主会话的 `tool_result`。

示例：

```json
{
  "status": "ok",
  "summary": "已抓取 3 个数据源，2 个成功，1 个超时",
  "artifacts": ["data/a.csv", "data/b.csv"],
  "errors": []
}
```

不要返回 raw log，不要返回完整推理过程。

### 3. 配置 session 写入策略

在 OpenClaw 的 session 配置里，可以设置 `write_filter`，只允许 `user`、`assistant`、`tool_result` 三类消息写入主 session，屏蔽内部 step、debug、subagent_thought。

```yaml
session:
  write_filter:
    allow: [user, assistant, tool_result]
    deny: [internal, debug, subagent_thought]
```

这样即使子 Agent 误写，也会被过滤层挡住。

### 4. 用 memory 分层处理跨 session 信息

如果子 Agent 确实需要保存长期结论，不要直接写主会话，而是写入 scoped memory 或 artifact store，再通过主 Agent 显式读取。主 session 只保存引用，不保存内容。

## 踩坑点

- **忘记 `inherit_context: false`**：子 session 会把主会话历史全量拷贝过去，子 session 瞬间爆掉；
- **图省事直接返回 raw transcript**：主 session 被塞进大量工具调用细节，token 消耗翻倍；
- **`parent_session_id` 没设置正确**：主 session 删除后子 session 成为孤儿，清理任务容易误删；
- **子 Agent 内部再调子 Agent**：session 层级太深，导致循环引用或权限继承错误。建议限制嵌套层级为 2；
- **只过滤消息类型，不限制 message size**：一个超大 `tool_result` 依然会污染上下文。需要加 `max_result_chars` 或 truncate。

## 可复用建议

- 让子 Agent 有明确的输入/输出契约，像函数一样设计；
- 主 Agent 调用子 Agent 时，只传必要上下文，不要“全量注入”；
- 设置子 session 的 TTL，任务结束自动归档，避免 session 表膨胀；
- 建一个 session 清理 job，定期删除 orphan session 和过期 debug 消息；
- 在 OpenClaw 日志里增加 `session_id` 维度，方便追踪哪些写入来自子 Agent。

## 总结

session 隔离不是简单地把子 Agent 扔到另一个线程，而是从 session 创建、写入策略、返回格式、记忆分层四个层面一起做。核心原则是：**子 Agent 的过程留在子 session，主 session 只接收结构化的结论和引用。** 这样主会话保持干净，Agent 推理更稳，token 成本也可控。

---

