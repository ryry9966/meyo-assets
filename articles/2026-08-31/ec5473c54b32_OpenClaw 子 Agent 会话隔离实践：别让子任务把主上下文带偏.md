---
title: OpenClaw 子 Agent 会话隔离实践：别让子任务把主上下文带偏
feedId: 35441
source: 综合讨论
publishedAt: 2026-08-31
---

# 背景

在 OpenClaw 的多 Agent 协作里，主 Agent 经常会把任务拆给子 Agent，或者通过插件、MCP 工具执行子任务。默认实现中，子 Agent 很容易复用主会话的上下文存储，导致中间推理、工具输出、错误堆栈全部回灌主 session。结果就是主上下文噪声增多、token 消耗上升，甚至出现“主 Agent 被某次失败子任务带偏”的情况。

这个问题在复杂自动化流程里尤其明显：一个子任务重试三次，三次的完整过程都留在主历史里；一个 MCP 工具返回了几十 KB 日志，主 Agent 后续决策时注意力全被这些无关信息占据。session 隔离不是框架自动保证的，需要主动做工程约束。

# 问题

具体来说，子 Agent 污染主会话通常有这几种表现：

- 子 Agent 的中间 steps 和工具调用日志进入主历史；
- 并发子任务共享 session id，消息交错；
- 子 Agent 失败重试的完整过程留在主上下文中；
- 插件/MCP 输出未经裁剪，一次性灌入大量文本；
- 主 Agent 后续推理被这些噪声干扰，甚至改变决策方向。

# 做法/步骤

## 1. 显式创建子 session，不继承父 session_id

不要使用默认构造。给每个子 Agent 分配独立的 session id，建议用命名空间前缀，例如：

```
{parent_session_id}:sub:{task_id}
```

这样 memory 和 history 天然分开。OpenClaw 中如果子 Agent 构造时不传 session 相关参数，很可能沿用父级，必须检查并显式覆盖。

## 2. 子 Agent 只回传结构化结果，不回传完整 history

定义裁剪后的返回结构，只保留必要字段：

```python
def run_isolated_subagent(task, parent_session_id):
    sub_id = f"{parent_session_id}:sub:{uuid4().hex[:8]}"
    sub_agent = create_sub_agent(session_id=sub_id)
    try:
        raw = sub_agent.run(task)
        return {
            "status": raw.status,
            "summary": summarize(raw.history),
            "artifacts": raw.artifact_refs
        }
    finally:
        sub_agent.close_session()
```

记住：直接 `return sub_agent.messages` 等于把子 history 带回主会话。

## 3. 只传上下文摘要，不传原始消息

如果子 Agent 需要参考主上下文的某些信息，不要直接把主 session 的消息数组传进去，而是生成一个简短的上下文摘要或提取关键变量。这样既保护了主会话的边界，也减少子 Agent 的初始 token 消耗。

## 4. 工具输出写入子 session

MCP 调用或插件执行时，把 returned content 存入子 session 的 memory，而不是主 session。这个需要在工具调用封装层处理，很多默认实现是直接回传主线程。

## 5. 任务结束显式清理

子任务完成后调用 `close_session()` 或等价的清理方法，释放 memory。如果有持久化需求，把关键产物写入工作区或对象存储，而不是留在消息流里。

# 踩坑点

- **默认继承**：很多框架子 Agent 构造时不传 session_id 会沿用父级。必须检查默认参数，不能想当然。
- **异步并发**：多个子任务若共用同一个子 session id，会消息交叉。每个 task 必须独立 id。
- **回传污染**：直接返回子 agent 的完整消息历史是最常见的污染源。要强制走裁剪函数。
- **memory namespace**：有些 memory store 按 agent id 或 session id 分区，如果没区分，子 Agent 初始化时会自动加载父历史，导致隔离失效。
- **清理不及时**：子 session 文件持续增长，尤其是插件输出大量日志。建议设置 TTL 或定期清理。
- **摘要过度压缩**：过度压缩会丢失关键错误信息，影响排障。保留 `error_trace` 前 N 行或关键堆栈。

# 可复用建议

- 封装一个统一的 `run_isolated_subagent` 工具函数，集中处理 session 创建、结果裁剪、清理。
- 给所有子 session id 加前缀，方便监控和排障。
- 定义 `SubAgentResult` 数据模型，只允许三个字段：状态、摘要、产物引用。
- 对工具输出设置截断阈值（例如 2000 字符），超出部分写文件，不在消息中传递。
- 增加集成测试：跑一个子任务后断言主 session 的消息数量不变。
- 监控主 session 长度，设置告警，避免静默爆上下文。

# 总结

子 Agent 的 session 隔离需要从三个边界做工程约束：**创建边界**（独立 session id）、**回传边界**（只回传结构化结果）、**清理边界**（显式 close 与截断）。核心原则很简单：子任务内部过程留在子 session，主会话只接收干净的摘要和产物引用。把这一点固化到代码规范里，才能避免子任务把主 Agent 带偏。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/882ade82a1d74f7f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/9dd8719ee9c9cbb0.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/142b9410d58960db.png)

