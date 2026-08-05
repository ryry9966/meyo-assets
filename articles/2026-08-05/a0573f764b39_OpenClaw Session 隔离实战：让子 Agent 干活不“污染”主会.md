---
title: OpenClaw Session 隔离实战：让子 Agent 干活不“污染”主会话
feedId: 31747
source: 综合讨论
publishedAt: 2026-08-05
---

## 背景：多 Agent 协作中的隐忧

在 OpenClaw 的多 Agent 编排中，我们经常会让主 Agent 向子 Agent 下发子任务，例如：

- 主 Agent 负责理解用户意图、规划分解；
- 子 Agent 专门执行代码生成、数据库查询、文件操作，或调用 MCP 工具。

理论上这是一个干净的“委托-返回”模式，但真正落地时会发现一个棘手问题：**子 Agent 的内部推理、工具调用链、甚至中间对话残渣，经常被带回主会话**。结果就是主 Agent 上下文被大量无关 token 撑爆，后续推理质量下降，甚至出现“幻觉式总结”——把子任务里才有的中间数据当作最终事实输出。

这在企业自动化场景中尤为要命：当子 Agent 调用 MCP 服务器拉取订单数据时，如果它的中间 SQL 查询、分页调试信息混入主会话，不仅浪费 token，还会泄露数据库视图结构。

## 问题拆解：为什么会污染？

根本原因不在于“有没有区分 Agent 实例”，而在于 **Session 的上下文字段是一块共享空间**。OpenClaw 内部默认的会话传递机制会把所有 Agent 的对话历史、工具调用消息、思考链（chain-of-thought）追加到同一个 `messages` 列表或类似结构中。即使你在代码里显式创建了子 Agent，只要没有做隔离，它就会往同一个 `Session` 对象里写东西。

具体表现为三种典型污染：

1. **对话历史污染**：子 Agent 的 system prompt、few-shot 示例、中间问答残留在主上下文中；
2. **工具消息污染**：子 Agent 调用的 tool_call / tool_result 消息被主 Agent 可见，后续规划会把工具错误当成自己的错误；
3. **状态残留**：子 Agent 通过 Session 级别的变量或 memory 写入的数据，被主流程无意中读到（尤其是 shared store 场景）。

## 做法：用 Session 分支实现“洁净室”子代理

OpenClaw 从 v0.7 开始提供了 **Session forking + isolated runner** 组合，专门解决这类问题。核心思路是：为子 Agent 创建一个从主 Session 继承必要上下文的新 Session，但在其内部执行时完全隔离，结束后只把提炼后的结果写回主 Session。

### 步骤一：从主 Session 派生一个“可遗弃”会话

不要直接复用 `current_session`，而是通过 `session.fork()` 创建一个子会话：

```python
child_session = current_session.fork(
    inherit_system_prompt=False,   # 不共享主 Agent 的 system prompt
    inherit_messages=False,        # 不把历史对话带入子 Agent
    inherit_tool_history=False,    # 关键：隔离工具调用记录
    metadata={"role": "subtask", "parent_id": current_session.id}
)
```

`fork()` 默认会拷贝全局配置、MCP 连接等资源，但通过参数可以精细控制继承内容。

### 步骤二：配置子 Agent 运行在子会话上

创建子 Agent 时，强制绑定上一步的 `child_session`，并指定 `runner="isolated"`：

```python
sub_agent = SubAgent(
    name="order_fetcher",
    session=child_session,
    runner="isolated",
    prompt=SYSTEM_PROMPT_ORDER,  # 专用于子任务的 system prompt
    tools=[mcp_order_tool]       # 仅授予必要工具
)
```

`runner="isolated"` 会确保子 Agent 在执行过程中产生的任何 message 都只写入 `child_session`，不会反向污染父会话。

### 步骤三：获取结果并安全销毁

等待子 Agent 完成，然后从 `child_session` 中提取你真正需要的信息，最后手动销毁子会话：

```python
result = await sub_agent.run(task)
# 只把最终结果写回主会话，通常是一段结构化摘要
current_session.add_message({
    "role": "assistant",
    "content": f"子任务 {sub_agent.name} 完成，摘要：{result.summary}"
})
# 销毁子会话，释放内存和 MCP 连接
await child_session.destroy()
```

注意：**千万不要**把 `child_session.messages` 整个合并回主会话，那样隔离就白做了。

## 踩坑点

### 1. MCP 连接未真正隔离

`fork()` 复制了 MCP 连接配置，但底层可能仍是同一个连接池。如果子 Agent 向 MCP 服务器发送了带状态的指令（例如设置 session 变量），主 Agent 后续调用同一 MCP 工具时可能读到脏状态。  
**解决**：在子 Agent 的工具函数包装中加入 `before_call` 重置 MCP 连接状态，或对需要隔离的 MCP 服务器使用独立端点实例。

### 2. 子 Session 销毁不彻底导致内存泄漏

如果在 `child_session.destroy()` 前有未取消的定时器或异步任务（例如流式日志），子会话虽然被 GC，但底层存储资源可能不会被立刻回收。  
**实践**：子 Agent 运行后显式调用 `child_session.close()`，并通过 `with` 上下文管理器确保清理：

```python
async with current_session.fork(...) as child_session:
    sub_agent = SubAgent(session=child_session, ...)
    result = await sub_agent.run(task)
# 退出时自动销毁
```

### 3. 错误把子 Session 的 tool 调用日志当作错误传播

默认情况下子 Agent 抛出的异常会携带 `child_session.messages`，如果主流程的异常处理不当，会把那些调试信息打印到主日志或塞回主会话。  
**建议**：在 `run` 外层捕获异常，构建干净的错误摘要返回主会话，断开原始上下文的引用。

## 可复用建议

- **最小继承原则**：`fork()` 时只继承资源（如 MCP 配置、认证 token），不继承任何对话和工具历史；  
- **结构化返回**：要求子 Agent 返回 JSON/结构化摘要，而不是把对话直接拼接回主会话；  
- **监控 Session 数量**：在长期运行的任务中，为 `fork()` 设置超时和最大子会话数，防止并发子任务打爆内存；  
- **与 MCP 工具搭配时的命名空间隔离**：利用 OpenClaw 的 `tool_namespace` 参数，为子 Agent 的工具调用加上前缀，避免工具结果被主 Agent 误读。

## 总结

OpenClaw 的 Session 隔离不是简单的“加点 context 限制”，而是一个需要从会话分叉、运行器隔离、销毁清理、错误传播四个维度完整设计的工程实践。正确实现后，子 Agent 就真像一个干净的工作间：进去的是任务描述，出来的是可用的结构化结果，不留任何脏脚印。对于需要频繁调用工具、访问外部数据的自动化流程，这套机制能显著提升主 Agent 的稳定性和输出的可信度。

---

