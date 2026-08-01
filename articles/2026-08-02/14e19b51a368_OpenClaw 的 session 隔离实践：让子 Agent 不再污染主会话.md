---
title: OpenClaw 的 session 隔离实践：让子 Agent 不再污染主会话
feedId: 31273
source: 综合讨论
publishedAt: 2026-08-02
---

## 背景

在 OpenClaw 构建多 Agent 协作时，经常会出现“主 Agent 调用子 Agent”的场景——主 Agent 负责总体编排，子 Agent 执行搜索、数据分析、文件改写等单一任务。一个常见的困扰是：子 Agent 的内部对话、工具调用记录、系统提示等一股脑涌进主会话，导致上下文爆炸、主 Agent 逻辑紊乱，甚至因 token 超限而任务中断。这种现象我们称之为**会话污染**。

OpenClaw 提供了灵活的 session 管理原语，我们可以通过实现子 Agent 的 session 隔离，让每个子任务在独立空间运行，只把最后的精华结果返回给主会话。

## 问题：共享会话的典型坏味

- **调试噪声干扰决策**：子 Agent 的中间 Thought 或 tool output 会进入主会话，主 Agent 可能误把这些内部信息当作用户指令，产生错误分支。
- **历史膨胀导致超限**：每轮子任务都追加几 KB 的上下文，多轮后 token 数轻松突破模型限制。
- **敏感数据泄露**：不同子 Agent 之间可能通过共享记忆意外暴露 API key、内部路径等敏感信息。
- **并发竞争**：当多个子 Agent 同时写入同一个 session 时，会出现消息乱序、状态覆盖等数据一致性问题。

根本原因在于，开发者习惯直接在主 session 上调用子 Agent，而没有为子 Agent 创建隔离的会话边界。

## 做法：子 Agent 独立 session 三步走

下面是一种经过验证的 OpenClaw session 隔离范式，可适用于各类子 Agent（无论是基于 `AutoAgent` 还是自定义 `Tool`）。

### 1. 创建独立子会话

OpenClaw 的会话由 `ChatSession` 对象管理。在触发子任务时，通过工厂方法新建一个 session，并指定唯一的 `session_id`，避免与主会话产生任何隐式引用。

```python
from openclaw import ChatSession

def create_child_session(parent_id: str) -> ChatSession:
    child_id = f"{parent_id}_child_{uuid.uuid4().hex[:8]}"
    session = ChatSession(
        session_id=child_id,
        model_config=parent_session.model_config,  # 复用模型配置
        max_tokens=2048,  # 限制子会话 token 消耗
        memory_backend=None,  # 关闭长期记忆，防止侧漏
    )
    return session
```

要点是**不要**将主 session 直接传递给子 Agent，也不要在子会话中使用主 session 的 `history` 对象。

### 2. 注入上下文并执行

通过一条“任务消息”将必要信息注入子会话，然后启动子 Agent。可以是用户消息形式，也可以是一条 `system` 指令。执行完毕只取最终回复，忽略中间所有内部步骤。

```python
async def run_isolated_sub_agent(parent_session, task_prompt: str):
    child = create_child_session(parent_session.session_id)
    # 注入仅包含任务描述的上下文
    child.add_message({"role": "user", "content": task_prompt})
    # 初始化子 Agent（配置为只使用该 session）
    sub_agent = SubAgent(session=child)
    final_response = await sub_agent.run()
    # 提取最终文本输出
    result_text = final_response.messages[-1]["content"]
    # 销毁子会话，释放资源
    child.close()
    return result_text
```

### 3. 回传精简结果

只把 `result_text` 作为一条格式化的消息追加到主会话中，例如：

```python
parent_session.add_message({
    "role": "assistant",
    "content": f"[子任务完成] {task_name}:\n{result_text}"
})
```

这样主会话始终保持干净，只保留了子任务摘要，避免了上下文污染。

更高级的做法是使用 **session bridge**：主会话通过临时消息传给子会话，子会话完成后再回写一条结构化响应，主 Agent 只需做一次路由即可知道子任务状态。

## 踩坑点实录

在实际工程中，以下几个坑需要格外留意：

1. **插件/工具的隐式 session 引用**  
   部分内置工具或 MCP 连接器会默认使用全局 session，例如 `get_current_session()`。在子会话中调用它们时，确保工具/插件的 `session` 参数被显式传入。必要时改造工具包装器，让它仅接受传入的 session，避免静默写入主会话。

2. **子会话溢出的 token 控制**  
   如果子任务自身需要多轮思考，很容易突破 `max_tokens` 限制。需要在外层捕获 `TokenLimitError` 并提前截断或降级，防止异常中止导致结果未返回。

3. **忘记关闭子会话**  
   长时间运行的服务会因未关闭的 session 造成内存和连接泄漏。建议使用上下文管理器自动回收：

```python
async with managed_child_session(parent_id) as child:
    # 执行子任务
```

4. **并发子 Agent 的 session ID 冲突**  
   如果不慎使用固定 ID，并发调用会相互覆盖。务必用 UUID 或原子计数器保证 ID 唯一。

5. **回传结果中夹带内部日志**  
   有些模型在输出时会不自觉地将思考过程一同输出。可以通过 post-processing 正则清理掉 `<thinking>` 或类似标记，或者在 prompt 中强调“只输出最终答案，不要包含任何中间步骤”。

## 可复用建议

- **封装 SubAgentRunner**：将创建、上下文注入、执行、清理和结果提取封装成一个标准的 `SubAgentRunner` 工具，主 Agent 只需调用它，避免了重复的 session 管理代码。
- **统一 Result Schema**：规定所有子任务返回一个包含 `status`, `summary`, `data` 字段的 JSON，方便主 Agent 解析和决策。
- **监控与告警**：对子会话的创建/关闭数量、平均 token 消耗、异常频率进行埋点，发现 session 泄漏或滥用及时报警。
- **测试隔离效果**：编写集成测试，对比开启隔离前后的主会话消息数量与 token 用量，确保隔离不会意外引入重复消息和无关历史。

## 总结

会话隔离是 OpenClaw 多 Agent 实践中的一项基础但关键的能力。通过为子 Agent 创建独立 session，严格控制上下文流动，我们能有效防止主会话被污染，显著提升系统的稳定性、可观测性和安全性。照着上述范式落地后，无论子 Agent 多复杂，主会话始终保持一张干净的“白板”，简洁、清晰、无限轮次可扩展。

---

