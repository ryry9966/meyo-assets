---
title: OpenClaw 子 Agent 的 session 隔离：别让子任务中间过程回流主会话
feedId: 33656
source: 综合讨论
publishedAt: 2026-08-18
---

## 背景

在 OpenClaw 里，主 Agent 调度子 Agent 处理长任务已经是很常见的模式。子 Agent 负责代码生成、批量文件处理、跨工具信息收集，主 Agent 负责编排和决策。早期我图省事，直接让子 Agent 复用主 session，结果很快遇到问题：主上下文里充斥着子任务的中间推理、工具调用输出、失败重试日志，token 成本快速上涨，主 Agent 也被无关信息带偏。

后来我把子 Agent 的 session 单独隔离，只让最终结果回流主会话，稳定性好了很多。这篇文章把实践过程整理出来，供有类似问题的同学参考。

## 问题：子 Agent 污染主会话的典型表现

- 主上下文快速膨胀，一次批量任务可能带入几万 token 的中间输出。
- 主 Agent 被子任务的错误重试日志干扰，做出错误决策。
- 多个子任务并行时，互相的消息混在一起，难以追踪来源。
- 敏感信息通过子 Agent 的临时文件路径或报错堆栈泄漏到主会话。
- 调试时无法分清哪些步骤发生在主会话，哪些发生在子会话。

核心矛盾是：子 Agent 需要上下文来完成任务，但主 Agent 只需要知道“这件事做完了没有、结果是什么”，不需要知道“过程中每一步怎么做的”。

## 做法：给子 Agent 建立独立的 session 边界

我在 OpenClaw 里通常分成四步做隔离。

**1. 为每个子任务创建独立 child session**

不要复用当前主 session 的消息列表。创建子 session 时绑定 parent_id，方便追踪父子关系，但消息历史完全独立。

```text
child_session = openclaw.session.create_child(
    parent_id=main_session.id,
    namespace=f"subtask:{task_id}",
    max_turns=20
)
```

**2. 主 Agent 只接收结构化回传**

让子 Agent 返回固定 JSON 结构，例如：

```json
{
  "status": "ok",
  "summary": "已完成 12 个文件的格式转换",
  "artifacts": ["/tmp/output/batch1.json"],
  "error": null
}
```

不要默认回传完整 transcript。OpenClaw 里有回传模式配置的话，我会显式设为 `summary` 或 `final`，避免全量消息合并。

**3. 工具和 memory 边界**

子 Agent 只挂载完成任务必需的工具。比如做文件处理，就不给它发消息通知、写主 memory 的权限。memory 写入走单独命名空间，或者直接设为只读。这样即使子 Agent 产生中间记忆，也不会污染主 Agent 的长期记忆。

**4. 异常回传策略**

子 Agent 失败时，只回传 `exit_code` 和简短错误摘要。完整堆栈和工具报错留在子 session 的 debug 日志里，主会话只记录“子任务 A 失败：依赖包安装超时，建议重试”。

## 踩坑点

**1. 忘记关闭子 session**

子任务结束后若不显式 close，子 session 可能仍在后台，审计日志里出现幽灵调用，甚至继续写入文件。我给自己定的规则是：子 Agent 的 finally 块里必须 close。

**2. 回传模式没配对**

有些框架默认会把子会话全部消息合并回父会话。只创建 child session 还不够，必须确认返回模式是 summary 或 final，否则隔离只做了一半。

**3. 共享 memory 插件造成隐形污染**

即使 message history 隔离了，如果子 Agent 和主 Agent 共用同一个 memory 插件，子 Agent 写入的长期记忆仍会影响主 Agent 后续决策。需要给子 Agent 单独 memory 表，或关闭写入权限。

**4. 并行子任务 session id 冲突**

不要用自增数字或过于简单的 task 简名作为 session id。多个子任务并行时容易撞车，导致互相覆盖。使用 uuid 或带时间戳的完整 id 更安全。

**5. 异常时全量堆栈回传**

工具报错堆栈往往很大，一旦全量回流主会话，几十万 token 就没了。需要提前在子 Agent 的异常处理里截断，只回传错误类型和可恢复建议。

## 可复用建议

- 封装一个 `spawn_isolated_agent` 工具，固定返回 JSON 契约，主 Agent 不再直接接触子 session 原始消息。
- 在 OpenClaw 配置中开启 session metadata 记录，便于追踪父子关系、创建时间、工具调用次数。
- 主 Agent 每次调用子 Agent 前打印预计输入/输出大小，避免大结果隐式回流。
- 给子 Agent 设置更低温度或专用模型，减少发散，同时降低中间过程的 token 消耗。
- 定期审计主 session 的消息来源，检查是否有子任务消息误入。

## 总结

session 隔离不是禁止子 Agent 使用上下文，而是让主会话只保留决策路径和最终结果，过程留在子会话里。这样主 Agent 更稳定，调试和成本也可控。在 OpenClaw 里做子 Agent 编排时，把它当成一个独立的短暂任务来管理，而不是主会话的一个“插件”，就能避免大部分污染问题。

---

