---
title: OpenClaw 的 session 隔离：子 Agent 怎么不污染主会话
feedId: 34818
source: 综合讨论
publishedAt: 2026-08-26
---

# OpenClaw 的 session 隔离：子 Agent 怎么不污染主会话

## 背景

用 OpenClaw 跑长任务时，主会话通常同时承担两类工作：一是规划与决策，二是调度子 Agent 执行具体子任务。常见做法是让子 Agent 直接在当前 session 里运行，把推理过程、工具输出、失败重试全部写回主会话。短期看省事，任务一多，主上下文就会被大量中间信息占满。

这类污染不是抽象问题：主会话的注意力会被子 Agent 的碎日志带偏；后续步骤容易重复引用无关的工具输出；一旦某个子 Agent 重试多次，错误堆栈和半成品内容会长期留在上下文里，影响后续规划。OpenClaw 本身不会自动隔离子 Agent 的 transcript，尤其是直接调用 agent loop 或共用消息通道时。

## 问题：共享 session 的三个污染来源

1. **上下文膨胀**：子 Agent 的完整 transcript 动辄几千 token，主会话只关心结果，却被迫承载过程。
2. **格式干扰**：子 Agent 的工具输出、JSON 片段、调试信息与主会话的规划文本混在一起，后续解析和判断都受影响。
3. **状态残留**：子 Agent 写入的临时变量、错误码、中间文件指针，如果不加命名空间，会覆盖或混淆主会话状态。

在 OpenClaw 的插件/MCP 调用里，这种污染更容易被放大：多个子 Agent 并发执行时，如果都往同一个 session 里写事件，主会话会变成事件流，而不是决策上下文。

## 做法：把子 Agent 当成函数，而不是对话成员

核心原则很简单：**子 Agent 使用独立 session，主会话只接收结构化结果，不接收过程 transcript。**

### 1. 为子 Agent 创建独立 session

不要复用 `parent_session_id`。每次调用子 Agent 时，在 session manager 里创建一个 `child` 类型的独立 session，并绑定父 session 引用，方便追踪和清理。

```python
child = await session_mgr.create_child(
    parent_id=current_session_id,
    ttl=1800,  # 子 session 最长存活时间
)
```

### 2. 定义回传结构

子 Agent 完成后，不要直接 `return transcript`。回传一个精简结构：

```python
{
    "status": "ok",
    "summary": "已完成用户列表清洗，去重 12 条，输出位于 artifacts/users.csv",
    "artifacts": ["artifacts/users.csv"],
    "errors": []
}
```

`summary` 由摘要器生成，限制在 300-600 token。`errors` 只保留脱敏后的错误类型，不回传完整堆栈。

### 3. 用 MCP 工具封装子 Agent

在 MCP server 内部实现 `run_subagent` 工具。OpenClaw 主会话通过工具调用触发，工具内部完成 session 创建、执行、摘要、销毁：

```python
async def run_subagent(task: str, context_snapshot: dict):
    child = await session_mgr.create_child(
        parent_id=current_session_id,
        ttl=1800,
    )
    try:
        transcript = await agent_loop.run(
            task=task,
            context=context_snapshot,
            session_id=child.id,
        )
        return {
            "status": transcript.status,
            "summary": summarizer.compress(
                transcript, max_tokens=600
            ),
            "artifacts": transcript.artifacts,
            "errors": sanitize_errors(transcript.errors),
        }
    finally:
        await session_mgr.archive(child.id)
```

主会话里只出现一行工具结果，不会出现子 Agent 的完整思考过程。

### 4. 需要读回的内容使用白名单

如果主会话确实需要某些中间产物，不要全文引入。只回传文件路径、关键指标、结构化字段。后续步骤通过 artifact 引用读取，而不是把文件内容塞进上下文。

## 踩坑点

- **回传全文**：直接把 `transcript` 或 `messages` 返回给主会话，等于没有隔离。
- **错误堆栈不脱敏**：完整异常信息进入主会话后，模型容易反复围绕错误展开，而不是继续任务。
- **子 session 不回收**：创建 child session 后不设置 TTL 或不归档，长时间运行后 session 表膨胀，恢复时也容易错乱。
- **共享 memory 不隔离**：如果子 Agent 和主会话共用同一个 memory 写入通道，子 Agent 的临时状态会污染主状态。需要按 session 或 namespace 隔离。
- **把进度事件写进上下文**：执行进度、工具调用日志应该走事件通道或日志系统，不要作为消息写入主 session。

## 可复用建议

1. 子 Agent 的返回结构固定为 `status / summary / artifacts / errors` 四字段。
2. 给 child session 设置 TTL，并在结束或异常时强制归档。
3. 摘要长度设置硬上限，超过就截断，不要依赖模型自觉压缩。
4. 通过 MCP 工具或插件接口调用子 Agent，而不是让主会话直接编排子 Agent 的消息。
5. 并发子 Agent 时，每个子 Agent 使用独立 session，最后合并结构化结果。

## 总结

OpenClaw 的 session 隔离不是“不让子 Agent 说话”，而是明确边界：主会话负责决策和状态，子 Agent 负责执行和计算。过程 transcript 留在子 session，回传主会话的只有结构化结果。这样主上下文始终保持在决策层级，子 Agent 的试错、重试和中间输出不会反过来影响规划。

在工程实践里，这比单纯调大上下文窗口更可靠，也更利于后续调试与审计。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/c3eb977eceb2f2e5.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/984d9f438b0522de.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/695cca52f28b5489.png)

