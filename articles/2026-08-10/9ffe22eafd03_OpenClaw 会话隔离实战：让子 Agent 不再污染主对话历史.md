---
title: OpenClaw 会话隔离实战：让子 Agent 不再污染主对话历史
feedId: 32365
source: 综合讨论
publishedAt: 2026-08-10
---

# OpenClaw 会话隔离实战：让子 Agent 不再污染主对话历史

## 背景：多 Agent 协作的上下文污染难题

在 OpenClaw 的实际工程里，我们经常需要把复杂任务拆解给多个子 Agent 处理。比如主 Agent 负责与用户对话、收集意图，然后把翻译、数据分析、文档生成等任务委派给专门的子 Agent。最“朴素”的集成方式是把子 Agent 的整个执行历史直接拼回主会话的消息列表里，这样主 Agent 就能“看到”子任务进展。但这种做法很快就会暴露一个严重的稳定性问题：子 Agent 的中间推理、内部指令、错误重试甚至工具调用日志，会像杂质一样污染主会话的上下文。

具体表现包括：随着对话轮次增加，主 Agent 逐渐忘记最初的任务指令，开始输出子 Agent 领域的术语；用户在主对话里看到的回答中突然出现了“I will now execute the Python script…”这类子 Agent 内部独白；当子 Agent 的任务失败时，它的错误堆栈和 debug 提示直接暴露给了最终用户。这些问题的根因，都是因为主客会话共用了同一个记忆存储，没有做上下文隔离。

## 问题本质：Session 是所有 Agent 的共享记忆

在 OpenClaw 中，每一个 Agent 实例并不直接持有对话历史，而是通过一个 `Session` 对象来读写消息。默认实现下，如果你没有显式分配 Session，多个 Agent 很可能会共享同一个全局 Session，或者通过管道串联时无差别地把消息往同一个消息列表里追加。这就意味着，子 Agent 的每一步思考、每一个工具调用的请求和响应，都会变成主 Agent 上下文的一部分，直到该上下文窗口被塞满，性能下降，回答质量崩塌。

更隐蔽的问题是，即使你给每个 Agent 创建了独立 Session，也可能在“如何传递必要信息”上犯错：把整个对话历史拷贝给子 Agent 虽然没有污染主会话，却可能让子 Agent 的上下文膨胀，同样影响其性能。因此，理想的隔离不仅要防止反向污染，还要控制正向传递的数据量。

## 做法与步骤：建立子 Agent 的隔离 Session 生命周期

我们可以在 OpenClaw 的插件或 Tool 执行层引入一个 **隔离 Session 工厂模式**，为每次子 Agent 调用生成一个短生命周期的 Session，并在调用完成后立即回收。以下是一个面向工程的可复现方案。

### 1. 明确主 Agent 与子 Agent 的边界

首先，在你的 OpenClaw 配置中，将主 Agent 声明为面向用户的入口，子 Agent 作为其可调用的内部服务。子 Agent 不应直接暴露给外部接口。如果使用 MCP 或其他工具协议，把子 Agent 封装成 MCP 工具是最简单的方式，因为 MCP 工具本身无状态，天然隔离。

### 2. 实现 IsolatedSessionManager

在你的项目中创建一个 `IsolatedSessionManager`，它负责：

- 提供一个工厂方法 `create_child_session()`：创建一个新的 Session，并可以设置最大生存时间（TTL，如 5 分钟），避免子 Agent 异常退出时遗留垃圾 Session。
- 提供 `destroy(session_id)` 方法，主动释放资源。
- 支持按需注入最简上下文：只把用户的原始问题、主 Agent 提炼的子任务描述和必要参数写入子 Session 的第一条消息，而不是复制整个主对话。

代码示意（Python 伪代码）：

```python
class IsolatedSessionManager:
    def __init__(self, store: SessionStore, ttl=300):
        self.store = store
        self.ttl = ttl

    def create_child_session(self, parent_id: str, instruction: str) -> Session:
        session = self.store.create(
            session_id=f"sub-{uuid4()}",
            parent_id=parent_id,
            ttl=self.ttl
        )
        # 仅注入任务指令，不注入历史
        session.add_message(SystemMessage(content=instruction))
        return session

    def destroy(self, session_id: str):
        self.store.delete(session_id)
```

### 3. 在子 Agent 调用时套用隔离会话

当主 Agent 需要调用子 Agent 时，通过一个内部 Tool 或 Plugin 来执行，步骤如下：

1. **输入提炼**：主 Agent 通过标准的 Tool Calling 机制生成一个包含子任务指令和必要参数的字典。
2. **创建 Session**：Tool 实现内部调用 `IsolatedSessionManager.create_child_session`，传入指令。
3. **执行子 Agent**：使用该子 Session 运行子 Agent，获取最终输出。通常可以调用 `sub_agent.run(instruction, session=sub_session)`，并确保子 Agent 的 `memory` 配置不指向任何持久化全局存储。
4. **返回纯结果**：从子 Agent 返回的最终消息中提取纯内容（如翻译后的文本、分析结论），丢弃子 Session 内的所有中间消息。
5. **销毁 Session**：在 `finally` 块中调用 `destroy`。

### 4. 调试与监控

为了在调试时能看到子 Agent 的完整执行过程，可以将子 Session 的消息流输出到独立的日志文件，而不是写回主 Session。OpenClaw 的钩子机制（如 `on_agent_start`, `on_agent_finish`）可以很方便地实现这一点。

## 踩坑点

- **坑1：子 Agent 需要主 Agent 的部分上下文，但复制了太多**
  解决办法：只传递任务描述和相关数据片段，由主 Agent 通过 Tool Calling 的入参显式提供。不要在子 Session 初始化时盲目复制最近的 N 条主对话消息。

- **坑2：忘记销毁临时 Session 导致内存泄漏**
  在高并发场景下，如果每次子调用创建 Session 但没有及时清理，Store（如 Redis 或内存）会迅速增长。必须使用 TTL 兜底，并在代码中保证 `destroy` 被调用。

- **坑3：子 Agent 的工具调用结果写回主 Session**
  检查子 Agent 的 `tool_config`，确保其工具的输出只写入子 Session。如果工具必须访问共享资源（如数据库），保证写入操作不产生副作用在主 Session 里。

- **坑4：多轮子 Agent 调用复用同一个子 Session 造成跨任务污染**
  每次独立的子任务调用都应使用全新的子 Session。即便是同一个用户请求中的连续子调用，也建议新建 Session，除非你能确定两次调用共享上下文是必要且安全的。

## 可复用建议

1. **封装为一个 OpenClaw 插件**：将上述 Session 管理逻辑封装成 `SessionIsolationPlugin`，在 Pipeline 的合适阶段自动注入。
2. **优先用 MCP 工具做无状态代理**：如果子 Agent 逻辑可以改造为纯函数式调用（输入->输出），直接将其包装成 MCP 工具服务，由主 Agent 通过 Tool Calling 调用，这样完全不需要手动管理 Session。
3. **配置可观测性**：在隔离会话的创建和销毁处埋点，输出日志包含 `parent_id` 和 `sub_session_id`，方便排查污染来源。
4. **设定资源上限**：对单个租户或单个主 Session 的最大并发子 Session 数做限制，防止恶意或异常调用耗尽资源。

## 总结

OpenClaw 的多 Agent 协作要走向生产级，上下文隔离是必须越过的坎。通过在子 Agent 调用时使用短生命周期的隔离 Session，并严格控制入参和出参，我们既能保持主对话的纯净，也能让子 Agent 聚焦于被委派的单一任务。这个模式工程化成本不高，却可以有效规避“记忆污染”引发的各类诡异行为。如果你的 OpenClaw 主 Agent 也正经历越聊越偏、泄露内部提示的困扰，不妨从 Session 隔离开始排查。

---

