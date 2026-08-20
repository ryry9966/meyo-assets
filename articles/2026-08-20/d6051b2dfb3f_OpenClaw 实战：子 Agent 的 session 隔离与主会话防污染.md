---
title: OpenClaw 实战：子 Agent 的 session 隔离与主会话防污染
feedId: 33900
source: 综合讨论
publishedAt: 2026-08-20
---

## 背景

OpenClaw 的主会话是一个持续累积的执行上下文：用户指令、工具调用记录、MCP 返回、插件 hook 日志都会写进去。引入子 Agent 处理长尾任务（比如批量搜索、代码生成、数据处理）时，如果子 Agent 直接跑在主 session 里，它的每一步推理、每一次失败重试、每一个中间工具输出都会变成主会话上下文的一部分。

这会让主 Agent 的注意力被大量过程信息稀释，token 成本上升，还可能出现敏感数据跨边界的问题。尤其是多个子 Agent 并发时，互相的中间结果混在一起，排查起来非常痛苦。

## 问题

实际用 OpenClaw 时，子 Agent 污染主会话通常表现为：

1. **上下文膨胀**：主会话从 3k token 涨到 30k，其中大部分是子 Agent 的 tool call 回显。
2. **指令稀释**：用户在开头给的关键约束被淹没在几十条中间日志里，主 Agent 开始“忘记”原始目标。
3. **并发互踩**：两个子 Agent 同时写 session，主 Agent 拿到的是交错在一起的过程数据，无法判断哪些是最终结果。
4. **边界模糊**：子 Agent 可能读到主会话里的敏感历史，或者把中间失败信息回传到不该出现的地方。

## 做法/步骤

### 1. 显式指定子 Agent 的 session 模式

OpenClaw 的 subagent 配置可以指定 session 隔离模式。以 YAML 配置为例：

```yaml
subagents:
  researcher:
    session:
      mode: isolated
      inherit:
        - user_goal
        - workspace_path
        - allowed_tools
      max_context_tokens: 8000
      return: summary
```

这里的关键是 `mode: isolated` 和 `return: summary`。子 Agent 在独立 session 里跑，主会话只接收最终的结构化摘要，而不是完整 trace。

### 2. 通过 MCP 工具封装子任务

如果 OpenClaw 版本支持 MCP 插件，可以封装一个 `subagent.run` 工具，让子 Agent 不直接接触主 session 存储。工具内部创建独立 session id，执行完成后只把 `final_message` 或 `summary` 写回主会话。

```python
# 伪代码：MCP 工具内部
def run_isolated(task, inherit_keys):
    sub_session = create_session(isolated=True)
    result = run_agent(task, session=sub_session, inherit=inherit_keys)
    close_session(sub_session.id)
    return result.summary
```

### 3. 控制结果回收粒度

不要使用 `return: full_trace`。除非你在调试，否则主会话只需要看到最终结果、关键引用和失败原因。中间步骤、重试记录、工具 schema 回显都不应该进入主上下文。

### 4. 验证隔离效果

每次跑完子任务后，记录主 session 的 token 增量。正常情况下，一个复杂子任务结束时主 session 只增加几百 token 的摘要，而不是几万 token 的过程数据。可以用 OpenClaw 的日志或 session stats 来确认。

## 踩坑点

### 隔离过狠，子 Agent 丢失必要上下文

一开始我把子 Agent 设成完全隔离，结果它不知道用户偏好和当前工作目录，生成的结果完全跑偏。后来改成 `inherit` 白名单，只传 `user_goal`、`workspace_path`、`allowed_tools` 这几个字段，问题解决。

**建议**：隔离不等于黑盒。把子 Agent 真正需要的最小上下文显式传进去。

### 工具侧泄漏

即使 session 隔离了，如果子 Agent 调用的 MCP 工具直接读写主 session 的存储，污染依然存在。比如某个工具会把结果 append 到主会话的 memory。这种情况下需要给子 Agent 设置工具白名单，或者让工具只读写独立副本。

### 并发写冲突

多个子 Agent 共享同一个 session store 时，隔离可能不彻底。每个子 Agent 应该有自己的 session id，而不是只靠 `mode: isolated` 标志。否则并发写入时仍可能串数据。

### fallback 回主会话

有些插件在隔离模式下找不到子 session 时会自动 fallback 到主会话。这很隐蔽。排查时如果发现主 session 里还是混入了子 Agent 的过程数据，先检查日志里有没有 fallback 相关记录。

### summary 截断丢失关键信息

子 Agent 返回的 summary 如果超过限制，可能被截断。这时主 Agent 拿到的摘要不完整，反而更糟糕。建议 summary 里保留 `source_ref` 或文件指针，而不是把完整内容塞进去。

## 可复用建议

- **建立 session 契约**：主会话只存“做什么”和“结果是什么”，子会话存“怎么做”和“中间过程是什么”。
- **给子 Agent 设置资源限制**：`max_tokens`、`max_turns`、工具白名单一起上，防止子任务失控。
- **用 metadata 标记来源**：回传的消息加上 `source: subagent:researcher`，方便追踪哪条结果来自哪个子 Agent。
- **监控 token 增量**：每次子任务后记录主 session 的 token 变化，设置告警阈值。如果增量异常，基本就是隔离失效了。
- **封装成 helper**：别每次手写隔离参数。定义一个 `runIsolated` 工具或配置模板，统一继承字段、返回策略和资源限制。

## 总结

OpenClaw 的子 Agent 隔离不是简单地把子任务丢进一个独立 session 就完事。它需要同时处理三件事：**上下文继承的最小化**、**结果回传的摘要化**、**工具和存储的边界控制**。做好这三点，主会话能保持精简，子 Agent 也能放开跑。踩过的坑大多来自隔离不彻底或隔离过度，关键是找到一个稳定的 session 契约，然后固化成工具或模板。

---

