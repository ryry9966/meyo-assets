---
title: OpenClaw 子 Agent 会话隔离实战：让主上下文不再被“污染”
feedId: 31434
source: 综合讨论
publishedAt: 2026-08-03
---

## 背景：一次子任务让主会话彻底跑偏

在 OpenClaw 多 Agent 编排中，我们经常会遇到这样的需求：主 Agent 在对用户输入进行规划后，把某个子任务委派给专门的下游 Agent（例如代码生成、数据分析、联网检索），然后根据返回结果继续推理。起初大家倾向于最简单的实现——直接在同一个会话上下文中调用子 Agent，让它的工具调用、思考链全部暴露在主流程里。

问题很快暴露：子 Agent 的长推理、多次工具调用、异常重试会把主会话的上下文撑得非常大，LLM 的注意力被稀释；更严重的是，子 Agent 的中间结论甚至错误推理会“感染”主 Agent 的下一步决策。一次失败的 SQL 重写流程可能让主对话带上错误的 schema 推断，导致后续回复彻底崩塌。这就是典型的“子 Agent 污染主会话”。

## 问题定位：共享 session 的三个致命缺陷

在 OpenClaw 默认的实现中，如果我们不给子 Agent 指定独立的会话边界，它的 execution trace 会全部追加到当前的 `Session` 对象中。这会带来三个工程层面的麻烦：

1. **上下文膨胀**：假设子 Agent 用搜索工具查看了 8 个网页、进行了 4 次反思，这些中间文本全变成主 session 的 memory，很快超出 token 窗口。
2. **指令串扰**：主 Agent 的 system prompt 和子 Agent 的 system prompt 同时存在于上下文，模型容易混淆角色，产出不符合预期的输出。
3. **错误传播**：子 Agent 如果中途被用户取消或者工具调用失败，其错误信息会残留在会话里，主 Agent 可能重复尝试修复根本不存在的问题，形成“幻觉循环”。

实际的案例中，有用户反馈“为什么我多加了几个插件之后，OpenClaw 开始自说自话地替我调用子 Agent 还反复纠正自己的错误”——追溯下去，就是主 session 里遗留了上一次子 Agent 留下的“我需要再试一次工具 X”的文本片段。隔离 session 的本质就是切断这种意外的信息继承。

## 做法：基于独立会话的“黑盒委派”

OpenClaw 提供了 `AgentSession` 的隔离能力，可以把子 Agent 包装成一个只返回最终结果的黑盒。核心做法分为三步：

### 1. 为子 Agent 创建依赖会话边界

不要在主 Agent 的 session 上下文里直接 `invoke`。正确的方式是利用框架的 `createSubSession()` 方法（或等价 API），为子 Agent 生成一个独立 session，里面仅包含子 Agent 的 system prompt 和当前需要处理的子任务描述。示例伪代码：

```python
sub_session = main_session.create_sub_session(
    agent_id="data_analyst",
    system_prompt=DATA_ANALYST_PROMPT,
    parent_message_id=None  # 不继承主会话的历史消息
)
result = sub_session.run(user_input="分析过去7天的销售趋势", max_turns=10)
# 仅把 result.final_output 作为一行文本写回主 session
main_session.add_message(role="tool", content=result.final_output)
```

关键点在于：子 session 的 `parent_message_id` 设置为 None，确保它从零开始构建上下文。

### 2. 严格控制信息回传粒度

子 Agent 的最终输出往往包含详细的推理过程，但主 Agent 只需要结论。可以在子 Agent 的 system prompt 中明确要求：“最后用 <final_answer> 标签包裹你要返回给主 Agent 的最终结果，不要附带任何推理过程。” 代码中解析标签，只把 `final_answer` 的内容写回主 session。这样主上下文始终是清洁的。

### 3. 捕获并转换异常

子 Agent 如果因工具不可用或超时而失败，它的错误信息绝不应直接塞进主 session。我们在 `except` 分支中，将错误信息转化为结构化的状态描述，如：“数据查询任务未能完成，原因：查询超时。请询问用户是否重试。” 这样主 Agent 就能根据状态而非原始报错进行决策。

## 踩坑点：三个容易被忽略的细节

- **子 Agent 的 token 配额**：独立 session 也有自己的上下文上限。如果子任务特别复杂（比如分析 10 万行日志），需要在子 session 上启用自动摘要或限制 `max_turns`，防止子 Agent 自身因为上下文溢出而无限重试。
- **工具调用状态的残留**：某些 OpenClaw 插件会将工具调用的中间缓存写到外部的 `workspace` 目录中。主、子 session 若共享同一个 `workspace`，子 Agent 下载的临时文件可能被主 Agent 误读。建议为子 session 分配临时目录，或使用 `with_temp_workspace()` 上下文管理器。
- **MCP 服务的 session 绑定**：当子 Agent 通过 MCP 调用外部服务时，如果服务端是以 session 为单位维护状态的（比如数据库事务），必须确保子 session 关闭后连接也随之清理，否则会导致连接泄漏。可以在 `finally` 块中显式调用 `sub_session.close()`。

## 可复用建议：形成子 Agent 包装规范

在实际持续交付的项目中，我们沉淀了一套小规范：

- 所有子 Agent 的调用都通过一个工厂函数 `call_sub_agent(agent_name, task, max_turns=8)` 进行，统一隔离逻辑。
- 子 Agent 的 system prompt 里强制包含 `response_format` 指令，要求仅返回 `{"status": "ok/failed", "data": ...}` 的 JSON。
- 主 session 只记录子 Agent 的调用摘要（任务类型 + 状态 + token 消耗），而不是任何中间输出。
- 建立 session 树的可视化监控，确保没有意外的主-子 session 继承关系。

这样既保留了多 Agent 协作的灵活性，又彻底消除了上下文污染的风险。

## 总结

子 Agent 会话隔离不是 OpenClaw 独有的问题，几乎所有多 Agent 编排都需要处理上下文边界。核心就两条：**信息传入时过滤，信息传出时裁剪**。把子 Agent 的思考过程关在黑盒里，只把干净的结论递交给主流程，既降低了 token 成本，也减少了由于历史残留导致的诡异行为。如果你的 OpenClaw 系统已经出现“Agent 反复自我纠正无意义错误”的症状，不妨从 session 树的结构入手排查，大概率能找到污染源。

---

