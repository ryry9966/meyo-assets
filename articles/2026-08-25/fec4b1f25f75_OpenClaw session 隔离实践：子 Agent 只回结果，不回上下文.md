---
title: OpenClaw session 隔离实践：子 Agent 只回结果，不回上下文
feedId: 34715
source: 综合讨论
publishedAt: 2026-08-25
---

## 背景

在 OpenClaw 里跑复杂自动化时，主 Agent 经常需要把子任务拆给子 Agent：抓取页面、处理文件、批量调用 MCP 工具链，或者执行一段独立的推理。子 Agent 本质上也是个完整 Agent，有自己的 reasoning、tool call、重试和错误处理。如果让它在主会话里直接展开，它的中间消息会全部写进主 session 历史。

这带来的不是“上下文长一点”的问题，而是主会话判断被污染：后续 planning 会看到子 Agent 的内部尝试、无关工具返回、失败重试等细节，主 Agent 容易被带偏。更隐蔽的是，共享 state 或 memory 时，子 Agent 的一次写入会反过来影响主会话后续行为。

## 问题表现

1. 主 session 历史里混入子 Agent 的中间输出，后续步骤被无关信息干扰。
2. 子 Agent 修改了 session 级变量或写入共享 memory，主会话状态被污染。
3. 子 Agent 的工具调用记录和事件流直接打到主会话前端，看起来就像主 Agent 自己在执行这些步骤。
4. 子任务失败重试时，消息重复追加，token 消耗暴涨，排障也很痛苦。

## 做法：每个子 Agent 一个独立 session

核心思路不是让子 Agent“少说点”，而是把它限制在独立边界里，只把最终结果回传。

### 1. 创建独立 session，用 parent_id 关联

不要复用主 session id。每个子任务创建一个新 session，通过 `parent_session_id` 建立归属关系。这样既能追踪来源，又不会共享主 session 的消息历史。

### 2. 只传任务载荷，不复制主历史

子 session 只需要知道“当前要做什么”，不需要主 session 的全部上下文。把主 session 的 messages 复制过去看似省事，实际上会把无关指令、历史修正甚至冲突的 system prompt 带进子任务，反而降低完成质量。

### 3. 子 Agent 输出结构化结果

子 Agent 的 system prompt 要明确要求只返回 JSON，禁止复述过程。主 session 只拿 `final_output`，不取 `child.messages`。

下面是一个伪代码示例，以 Python client 为例，实际使用时以你所用版本为准：

```python
import uuid

def run_subagent(main_session, task_name, task_input):
    child_id = f"sub:{main_session.id}:{task_name}:{uuid.uuid4().hex[:8]}"

    child = client.create_session(
        session_id=child_id,
        parent_session_id=main_session.id,
        system=(
            "You are an isolated sub-agent. "
            "Return only JSON: {\"status\":\"ok\",\"summary\":\"...\",\"artifacts\":[]}. "
            "Do not include intermediate reasoning."
        ),
        max_turns=8,
        max_tokens=2000,
        timeout_seconds=120,
    )

    try:
        result = child.run({"task": task_input})
        # 只把结构化结果写回主会话
        main_session.add_tool_result(
            tool_name=task_name,
            content=result.final_output,
        )
    finally:
        client.close_session(child.id)
```

### 4. 严格清理

子 session 执行完后，无论成功失败都要在 `finally` 中关闭。否则循环任务会留下大量空 session，占用资源，后续也难以审计。

## 踩坑点

- **把主 session 完整历史复制给子 session**：以为“上下文越全越好”，实际上会引入不相关指令和冲突信息，子 Agent 更容易跑偏。
- **只隔离对话，不隔离 memory**：共享 memory 写入会让子 Agent 的知识污染主 session 的长期记忆。应使用独立 memory namespace。
- **事件流没有按 parent_id 过滤**：子 session 的事件如果打到主 session 的 SSE/WebSocket 通道，前端会把子 Agent 的中间过程渲染成主会话内容。需要在事件消费侧做过滤。
- **只取 final_output 但没校验 schema**：子 Agent 返回自然语言或半截 JSON，主 session 解析失败后重试，旧 child session 又没关，导致 session 堆积。
- **忽略 parent_id 追踪**：出问题后无法定位是哪个子任务产生的，排障成本很高。

## 可复用建议

- 封装 `spawn_subagent(name, task, context, schema)` helper，统一创建、执行、校验、关闭流程。
- 子 Agent 的 system prompt 固定输出 JSON，并要求“不要复述过程，只给结果”。
- 子 session id 带上前缀 `sub:{parent_id}:{task_id}:{ts}`，方便 grep 日志。
- memory 使用独立 namespace，例如 `child_{child_id}_memory`，返回时只把必要结果写入主 memory。
- 主 session 侧只暴露一个 `run_subagent` 工具，内部完成隔离，不向外暴露子 session。
- 每次运行后保存 session manifest，记录 parent_id、child_id、task、status、tokens，便于成本审计。
- 做一次“污染测试”：调用子 Agent 后，检查主 session 最后 N 条消息是否只包含 tool result，没有子 Agent 的内部 reasoning。若有，说明边界还没关好。

## 总结

session 隔离不是把子 Agent 藏起来，而是给它明确的边界：独立 session id、最小上下文、只回结构化结果、显式清理。做好之后，主会话保持干净，子任务可以并行或重试，错误不会反向污染主 Agent 的 planning。对 OpenClaw 这类把 session 作为一等公民的框架来说，这条边界越早建立，后续自动化越稳。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/064daced1613c7fe.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/03a7edcf15bf7c43.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/4f3a62d2fc354526.png)

