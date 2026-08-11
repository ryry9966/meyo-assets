---
title: 子Agent 不污染主会话：OpenClaw Session 隔离的正确打开方式
feedId: 32512
source: 综合讨论
publishedAt: 2026-08-11
---

## 背景

在 OpenClaw 多 Agent 系统中，主 Agent 经常需要把计算密集、有副作用或需要重试的子任务委派给子 Agent 执行，例如调用沙箱中的代码、触发外部 MCP 工具链，或者完成一次需要多轮推理的复杂数据抽取。

如果不做任何处理，子 Agent 在完成任务时产生的全部推理步骤、工具调用细节、中间错误甚至调试输出，都会直接回流到主 Agent 的会话历史中。这会带来三个核心问题：

- **上下文膨胀**：大量无关 token 挤占主会话窗口，导致关键信息被截断，或直接推高模型调用成本。
- **信息污染**：子 Agent 的内部推理可能泄漏 prompt 细节、中间结果，干扰主 Agent 的决策逻辑。
- **容错耦合**：子 Agent 的重试、异常信息混入主历史后，主 Agent 可能会错误地重复相同动作，甚至因为看到异常文本而偏离原始任务。

OpenClaw 在其会话管理抽象中提供了 **Session 隔离** 能力，可以为每个子 Agent 创建独立执行上下文，主 Agent 只接收经过约定的最终结果。下面从工程视角展开这一模式的实施路径。

## 问题场景还原

假设一个电商客服 Agent 的主流程如下：

1. 接收用户查询订单状态。
2. 主 Agent 判断需要调取物流接口，但不直接调用。
3. 通过 `logistics_lookup` 工具，委派给一个子 Agent 进行处理。
4. 子 Agent 内部需要调用 3 个 REST API，处理重试、错误恢复，最终返回结构化结果。

如果使用默认的“内联”方式，子 Agent 的思考过程会像这样被写入主会话：

```
<assistant> 让我调用 logistics_lookup 工具...
<tool_result> [子Agent] 正在调用承运商A API，返回 429，等待 2 秒重试...
[子Agent] 重试第 2 次成功，获取运单号，解析中...
[子Agent] 预检地址格式失败，切换备用接口...
[子Agent] 最终提取结果: {"status":"shipped","eta":"2025-06-15"}
```

这些中间状态在主 Agent 眼里都是噪音。更糟的是，一旦子 Agent 产生较长的错误堆栈，主 Agent 的 prompt 中会混入大量技术细节，极易引发幻觉或错误的下游动作。

## 做法与步骤

OpenClaw 的 session 隔离通过 `SpawnAgent` 的 `session_policy` 参数实现。核心原则是：子 Agent 运行在独立 session 中，仅通过约定的终结信号返回精简后的结果。

### 1. 定义独立子 Agent

子 Agent 本身是一个普通的 Agent 定义，但它的输出应为单一结果，不依赖主会话历史。

```yaml
# agents/logistics_sub.yaml
id: logistics_sub
model: openai/gpt-4o-mini
system_prompt: |
  你是物流查询助手。根据输入的运单号，依次调用查询、轨迹、地址校验 API，
  如果任意步骤失败最多重试 2 次。最终以 JSON 格式返回结果，不要输出其他文字。
tools:
  - mcp: courier_api
  - mcp: address_validator
```

### 2. 在主 Agent 中以隔离模式生成子 Agent

主 Agent 配置中的工具通过 `spawn_agent` 生成子 Agent，并显式开启隔离：

```yaml
# agents/main.yaml
tools:
  - type: spawn_agent
    agent_id: logistics_sub
    session_policy: isolated   # 关键参数
    timeout: 30s
    input_mapping:
      tracking_number: "$input.tracking_number"
    output_template: |
      物流查询结果：
      {{ result.output }}
```

当主 Agent 调用 `logistics_lookup` 工具时，OpenClaw 会：

- 为子 Agent 创建一个全新的 `Session`，初始消息仅为子 Agent 的 `system_prompt` 和携带输入的用户消息。
- 子 Agent 的所有推理、工具调用、错误产出都在该独立 Session 中完成。
- 子 Agent 执行结束后，`openclaw` 仅提取 `Session.final_output`（即最后一条 assistant 消息），放入主 Agent 的工具结果中，而不是整个历史的拼接。

### 3. 与 MCP 配合时的额外隔离

如果子 Agent 使用的 MCP 服务器是全局共享的，仍然可能出现工具调用记录污染主会话的问题。最佳实践是为子 Session 创建 **临时 MCP 上下文**：

```yaml
mcp_handling: per_session   # 或 isolated
```

这样即使同一个 MCP 服务，不同 Session 间工具调用的结果也不会互相可见，进一步保证主会话清洁。

### 4. 监控与调试

隔离之后，主 Agent 的会话日志里看不到子 Agent 的详细过程。为了问题排障，需要显式记录子 Session 的 ID：

```python
# 在主 Agent 代码中获取子 session 引用
sub_record = spawn_agent.execute(params)
logger.info(f"子会话ID: {sub_record.session_id}, 耗时: {sub_record.duration}")
```

后续可通过 OpenClaw 的 Session Inspector 或日志系统直接检索该 ID 查看完整推理过程。

## 踩坑点

在落地过程中，有几个容易忽略的问题：

- **上下文断连**：子 Agent 被隔离后无法访问主会话中的用户偏好、前置对话。必须通过 `input_mapping` 把必要信息显式传递进去。一个常见错误是，主 Agent 假设子 Agent“知道”当前用户是谁，导致子 Agent 使用错误参数。在设计子 Agent 时，明确它的输入契约，做到无状态。

- **超时与 Hang 住**：如果子 Agent 进入死循环重试，独立 Session 又不会向主 Agent 实时反馈，主 Agent 可能会一直等待工具返回。必须设置 `timeout`，并配置 `fallback`（返回固定错误提示，例如“暂时无法查询物流”），防止整个工作流卡死。

- **MCP 连接泄漏**：使用 `per_session` 模式的 MCP 连接，如果子 Session 异常退出未被正确清理，可能导致长连接或凭证残留。务必增加 Session 生命周期的清理钩子，并监控未关闭的 MCP 连接数。

- **结果结构不统一**：子 Agent 的任务是返回可解析的结果。但模型有时会在 final_output 中多输出一句注释。建议在子 Agent 的提示词中严格限制输出格式，并在主 Agent 侧增加后处理校验（如尝试 JSON 解析，失败则视为异常）。

## 可复用建议

基于以上实践，整理一份可复用的隔离策略清单：

- **划分敏感边界**：凡是调用外部系统、存在副作用或需要多轮重试的子任务，默认走 isolated session。
- **设计干净的输入/输出协议**：子 Agent 的输入用 JSON Schema 约束，输出用类似 `{"ok": true, "data": {...}, "error": null}` 的包裹格式。主 Agent 只依赖 `output_template` 提取的信息。
- **兜底与告警**：为每个隔离子 Agent 设定最大执行时间，超时后触发告警，同时将子 Session 日志标记为 `aborted`，便于事后分析。
- **独立成本计量**：子 Session 单独累计 token，可以精确核算不同子任务的调用成本，也方便做降级策略（如当子 Agent 消耗超过 2000 token 时，降级为更简单的方案）。
- **利用 Session 快照**：对于需要审计的关键操作（如修改订单状态），把子 Session 的快照持久化，主 Agent 的工具结果中仅存放快照 ID，而不是内容，既隔离了信息又保有可追溯性。

## 总结

OpenClaw 的 session 隔离并不是“让子 Agent 消失”，而是提供了一条清晰的执行边界。它把过程藏在幕后，只让经过提炼的结果回到主 Agent 的视野。这不仅能显著抑制上下文膨胀、降低模型被干扰的风险，也推动了多 Agent 系统的模块化设计。

在复杂的 Agent 工作流中，**把噪音留在子会话，把决策留给主会话**，是一个低成本、高收益的工程原则。配合合理的超时、兜底和协议设计，隔离模式能让你更放心地把计算压力交给子 Agent，而不必担心主逻辑被污染。

---

