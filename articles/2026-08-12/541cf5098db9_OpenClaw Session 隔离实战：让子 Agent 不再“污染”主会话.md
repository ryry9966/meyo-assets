---
title: OpenClaw Session 隔离实战：让子 Agent 不再“污染”主会话上下文
feedId: 32744
source: 综合讨论
publishedAt: 2026-08-12
---

## 背景

在基于 OpenClaw 的多 Agent 系统中，主 Agent 负责理解用户意图、拆解任务，并按需将子任务委派给更专精的子 Agent。常见模式如“主调度 + 子执行”：一个面向用户的助手 Agent，背后挂着代码解释器、数据分析、网页抓取等 worker。任何一个子 Agent 在执行过程中，都会产生大量的中间推理、工具调用、对话轮次。

如果没有任何隔离机制，这些子 Agent 的内部“自言自语”会直接混入主会话的消息流里。后果有三：

1. **上下文爆炸**：无用的中间步骤挤占 token 预算，很快触发模型上下文限制；
2. **决策误导**：主 Agent 可能会把子 Agent 的工具返回、调试信息当成用户的新输入，做出错误判断；
3. **串扰风险**：在共享 memory 的场景下，不同会话的历史可能互相污染。

本篇文章梳理我在实际项目中使用 OpenClaw 管理子 Agent 时的 session 隔离方案，以及踩过的几个坑。

## 问题复现

假设主 Agent 需要调取一个“财务分析”子 Agent，它内部会依次调用 API 拉取数据、用 Python 计算、再生成摘要。如果不加以控制，子 Agent 的完整对话历史：

```
[子Agent] 正在理解任务...
[子Agent] 调用工具 get_finance_data
[工具返回] { ... 大段 JSON ... }
[子Agent] 数据获取成功，开始计算...
[子Agent] 计算结果：净利润增长12%...
```

所有这些消息都会被追加到主会话里。主 Agent 再看到这一串，经常出现“嗯？用户让我分析数据了吗？”的错觉，或者试图逐条解释。

## 实施步骤：在 OpenClaw 中做 Session 隔离

OpenClaw 提供了 `session_isolation` 配置项，作用于每个子 Agent 定义。核心思路是：**子 Agent 启动时创建独立 session，执行完成后只将最终答案“注入”主会话，中间过程全部丢弃。**

### 1. 配置子 Agent

在子 Agent 的 YAML 定义中增加：

```yaml
agent:
  name: finance_analyst
  type: llm_agent
  session_isolation: true        # 开启隔离
  max_turns: 6                   # 限制轮次，防止死循环
  return_mode: final_answer      # 只返回最终答案，而非完整消息
```

其中 `return_mode` 可以选：

- `full_history`：默认，返回所有消息（不推荐）
- `final_answer`：仅返回符合终止条件的最后一条回复
- `aggregated`：聚合所有工具返回和文本输出并压缩

### 2. 主 Agent 调用方式

主 Agent 通过工具或子流程调用子 Agent，OpenClaw 内部会为每次调用生成一个新的 `session_id`。伪代码逻辑：

```python
result = main_agent.call_subagent(
    name="finance_analyst",
    input_prompt="请分析Q3毛利率变化的原因",
    session_parent=current_session_id,
    isolated=True
)
```

此时主会话里只会新增一条消息：

```
[助手] 财务分析结果：Q3毛利率下降由于原材料成本上升，具体...
```

### 3. MCP 工具封装 (补充方案)

如果你的子 Agent 作为 MCP 工具提供，隔离天然更好：MCP 的 tool call 模式本身就是无状态的，服务端一次调用返回最终结果，不保留任何历史。只需注意在 tool 实现中不要主动加载主会话的 memory。

## 踩坑实录

### 坑1：共享 Memory 后端未做命名空间划分

初期我把所有 session 的 memory 都存到同一个 Redis DB，子 Agent 的 session key 是按 `session:sub_agent_id` 生成的。当并发调用同一子 Agent 类型时，不同主会话触发的子任务可能共享同一个 memory，导致数据串位。

**解决**：让 `session_id` 加入主会话标识和调用序号，如 `main_session123_finance_1`，并设置 TTL，任务完成主动清理。

### 坑2：异常堆栈原样返回

子 Agent 执行中出现 Python 异常，`final_answer` 没有做好 catch，把整个 traceback 当成最终答案传回主会话。用户看到一段“File xxx line 42 …”直接懵掉。

**解决**：在 `final_answer` 提取逻辑中增加异常过滤器，若检测到未格式化的异常文本，转为结构化错误信息：“子任务执行错误：[简要原因]，请联系管理员”。

### 坑3：插件自动注入上下文

OpenClaw 的一些插件（例如 Knowledge Retriever）会默认从“当前全局上下文”拉取信息，即使子 Agent 开启了隔离，插件内部可能还是读取到了主会话的历史片段，导致隔离不彻底。

**解决**：审查每个启用插件的 `context_scope` 设置，将其限定为 `session_scoped` 而非 `global`，必要时在子 Agent 配置中显式置空。

### 坑4：token 预算难以控制

虽然丢弃了中间过程，但子 Agent 内部仍会消耗大量 token，可能导致整个主会话超出上限。 **建议**：为主 Agent 和子 Agent 分别设定 `max_tokens_per_call` 和 `max_context_tokens`，通过 OpenClaw 的 `token_budget` 策略在调度层控制。

## 可复用的工程化建议

1. **子 Agent 设计为无状态函数**：输入就是一段完整的 prompt，输出就是一段确定的答案，不依赖之前调用的记忆。
2. **监控与日志**：在日志中同时打印 `parent_session_id` 和 `child_session_id`，便于追踪子任务归属。
3. **降级策略**：如果子 Agent 超时或 token 耗尽，回退到预设的默认回复，而不是让主 Agent 的整个会话挂掉。
4. **测试隔离效果**：写一个集成测试，主 Agent 连续调用两次相同子 Agent，断言第二次调用看不到第一次内部对话的任何信息。
5. **MCP 优先**：对于可以标准化为工具的子任务，直接用 MCP 暴露，无需在 OpenClaw 内再封装一层 Agent，隔离更彻底，调试更简单。

## 总结

Session 隔离是复杂 Agent 系统从“能跑”到“稳定可维护”的关键一步。在 OpenClaw 中，通过开启 `session_isolation` + 返回模式控制，可以有效避免子 Agent 污染主会话。但要注意 memory 命名空间、异常处理、插件上下文作用域等细节。最重要的是——保持子 Agent 简洁、短生命周期，让它真的像一个工具而非另一个会聊天的角色。

---

