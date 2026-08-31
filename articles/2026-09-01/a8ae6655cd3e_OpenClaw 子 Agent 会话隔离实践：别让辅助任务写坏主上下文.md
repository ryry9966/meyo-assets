---
title: OpenClaw 子 Agent 会话隔离实践：别让辅助任务写坏主上下文
feedId: 35624
source: 综合讨论
publishedAt: 2026-09-01
---

## 背景

在 OpenClaw 里跑多 Agent 协作时，最容易被低估的问题不是子 Agent 能力不够，而是它会悄悄污染主会话。一个典型的场景是：主 Agent 为了完成一个复杂任务，拉起子 Agent 去做搜索、代码生成或 MCP 工具调用。子 Agent 执行过程中产生大量中间输出——工具返回的原始 JSON、重试日志、调试信息、甚至一次失败尝试的完整堆栈——如果这些内容都写回主会话，主上下文很快就会变得臃肿且难以维护。

主上下文是稀缺资源。一旦被无关信息占据，主 Agent 的注意力会漂移，后续推理质量下降，token 成本直线上升，调试时也很难定位关键信息。更隐蔽的问题是，如果子 Agent 与主会话共享状态或插件写入权限，子 Agent 的副作用可能直接改掉主会话里的关键变量，造成难以复现的异常。

OpenClaw 提供了子 Agent 的 session 隔离能力，但默认配置并不总是足够安全。下面梳理一套工程上可落地的做法。

## 问题定位

在 OpenClaw 中，子 Agent 污染主会话通常来自三个路径：

1. **直接消息写入**：子 Agent 的完整执行日志、工具输出、错误堆栈被作为普通消息追加到主会话。
2. **共享状态/内存**：子 Agent 默认继承主会话的 memory 或 state store，修改后反过来影响主 Agent。
3. **插件副作用**：子 Agent 调用的插件如果绑定在父级 context 上，插件产生的写入或事件通知会直接进入主会话。

这三条路径不堵住，单纯限制子 Agent 的 token 数量只是治标。

## 做法 / 步骤

### 1. 显式启用子 Agent 独立 session

不要创建子 Agent 时省略会话配置。以 OpenClaw 的常见用法为例，需要在创建子 Agent 时指定独立 session：

```python
child = await openclaw.create_subagent(
    task=...,
    session_scope="isolated",  # 关键：不要用 "inherit"
    inherit_memory=False,
    return_mode="summary",
)
```

如果你的 OpenClaw 版本使用 `session_id` 控制，则为每个子 Agent 生成独立 session id，并确保主会话只接收最终返回值。

### 2. 限制子 Agent 的输出边界

子 Agent 内部可以随便折腾，但出口必须收敛。建议在子 Agent 的 system prompt 中明确：

- 工具输出只允许在内部消化；
- 最终只返回一段结构化摘要或 JSON；
- 禁止返回完整工具响应原文。

同时配置 `max_internal_iterations` 和 `max_output_tokens`，防止子 Agent 自行扩展输出范围。

### 3. 强制结果走结构化返回，而不是消息流

在 OpenClaw 中，如果子 Agent 通过 `send_message` 或类似机制主动向主会话推送消息，隔离就会失效。应当将子 Agent 结果封装为返回值，由主 Agent 显式读取：

```python
result = await child.run()
main_context.add("child_result", result.summary)  # 只取结构化摘要
```

避免让子 Agent 直接持有主会话的 writer。

### 4. MCP 工具单独实例化

如果子 Agent 需要调用 MCP 工具，不要在子 Agent 里复用主会话的 MCP client。MCP client 通常带有会话状态和事件回调，共用会把子 Agent 的工具调用生命周期混入主会话。正确做法是为子 Agent 单独创建 MCP client，并在子 Agent 结束时关闭连接。

```python
child_mcp = await create_mcp_client(server_config)
child = await openclaw.create_subagent(
    task=...,
    mcp_client=child_mcp,  # 独立实例
    session_scope="isolated",
)
```

### 5. 错误处理只回传错误码和简短原因

子 Agent 失败时，最容易污染主会话的就是完整 stack trace。在子 Agent 的异常处理外层包一层，把异常转换为结构化错误对象：

```python
try:
    return await child.run()
except Exception as e:
    return {
        "ok": False,
        "error_type": type(e).__name__,
        "brief": str(e)[:200],
    }
```

主会话只看到简洁的错误摘要，而不是几千 token 的堆栈。

## 踩坑点

**插件默认绑定父级 session**  
即使子 Agent 自身 session 隔离了，某些插件在初始化时会注册到父级 context 的事件循环上。插件产生的进度通知、回调日志仍可能绕过隔离直接进入主会话。排查方法是打开 OpenClaw 的 tracker，观察主会话中是否出现来自子 Agent 的插件事件。如果是，需要在子 Agent 内重新初始化插件，或者改用不依赖父级事件的插件版本。

**`inherit_memory` 默认值可能为 true**  
有些 OpenClaw 版本为了方便，子 Agent 默认继承父级 memory。这就等于共享了状态。务必显式设置 `inherit_memory=False`，并确认子 Agent 使用的 state store 是独立命名空间。

**并发子 Agent 消息交错**  
多个子 Agent 并发执行时，如果每个子 Agent 都往主会话写一点中间结果，消息顺序会乱，主 Agent 难以还原上下文。解决方法是所有并发子 Agent 都只返回最终结果，由主 Agent 统一收集后再做合并或排序。

**摘要阈值设置过高**  
如果配置了 `return_mode="summary"`，但摘要阈值设置成几千 token，那跟没隔离差不多。根据主上下文预算倒推，单个子 Agent 返回给主会话的摘要建议控制在 300–800 token 以内。

## 可复用建议

把子 Agent 当成“无状态函数”来设计：输入明确、输出结构化、不修改外部状态。所有需要持久化的结果，由主 Agent 统一写回主会话或外部存储。

在团队里可以形成几条硬约定：

- 子 Agent 禁止直接写主 session；
- 子 Agent 输出必须是 JSON 或固定 schema；
- 子 Agent 的 MCP client 必须独立实例化；
- 主会话只接收 final result，不接收过程日志；
- 每个子 Agent 任务结束必须关闭自己的 session 和连接。

另外，建议定期用 OpenClaw 的 session 分析工具查看主会话 token 分布。如果发现某类子任务贡献了异常高的 token，就把它改成更严格的摘要模式或独立 session。

## 总结

OpenClaw 的 session 隔离不是开关一开就完事，它需要你在创建子 Agent、配置插件、处理错误、设计返回格式时都保持边界意识。真正有效的隔离，是让子 Agent 在内部自由执行，但只把最精简的结果交还给主会话。这样主上下文才能保持干净，主 Agent 才能持续做高质量的决策。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/7bc6a9bb9b2bd6c6.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/226187e433959f0b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/726cee392744b4e5.png)

