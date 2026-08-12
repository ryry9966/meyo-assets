---
title: OpenClaw 子 Agent 隔离实战：如何让任务委派不污染主会话上下文
feedId: 32822
source: 综合讨论
publishedAt: 2026-08-12
---

## 背景
多 Agent 协作已经成为复杂自动化系统的刚性需求。OpenClaw 的主会话承载着用户与主 Agent 的完整对话、思考链以及各种工具调用的结果。当主 Agent 将子任务委派给子 Agent 时，默认行为是让子 Agent 共享主会话的全部历史，这看似方便，实则相当于给一个专注的小助手塞满了整间会议室的聊天记录。

在大上下文窗口普及的今天，大家容易忽略一个事实：**无关信息对推理质量的影响远比 token 消耗更隐蔽**。子 Agent 本应精确执行“查询库存”或“解析日志”这类单一任务，却被迫解析前面三十轮的闲聊和调试信息，最终导致延迟增加、指令遵循度下降，甚至把上游闲聊内容错误地编织进自己的输出，反向污染主会话。

## 问题表现
一个典型的翻车现场：
- 主会话已包含用户与 Agent 关于写诗、代码审查的多轮交互；
- 主 Agent 调用子 Agent “校验订单 9823 的发票信息”；
- 子 Agent 返回了发票状态，但同时在回复的末尾加了一句押韵的对联——因为它看到了主会话里的写诗指令。
- 主 Agent 很可能把这个“创意”回复原样转发给用户，破坏严肃场景。

更隐蔽的问题是：当主会话中遗留了大量“工具调用失败 / 重试”的噪音，子 Agent 会浪费注意力在这些与当前完全无关的调试信息上，导致本应一次成功的任务出现推理分叉。

## OpenClaw 的 session 隔离机制
OpenClaw 在子 Agent 配置层提供了 `isolated_session` 开关。开启后，每次调用子 Agent 不再分配主会话的全部消息列表，而是创建一个仅包含以下内容的全新会话：

- 子 Agent 自身的 `system_prompt`；
- 通过 `pass_variables` 显式传入的变量（被拼接为单条或多条用户消息）；
- 若开启 `include_tool_history`，则仅携带调用链路中必要的工具结果（默认关闭）。

```yaml
# sub_agents/inventory_checker.yaml
agent:
  name: InventoryChecker
  model: claude-sonnet-4-20250514
  isolated_session: true
  system_prompt: >
    严格按以下格式返回库存信息：{ product_id: string, stock: number }。
    不要输出任何额外内容。
  pass_variables: ["sku_id", "warehouse_zone"]
  include_tool_history: false
```

主 Agent 在调用时只需传递最小变量集：

```python
result = await claws.call_sub_agent(
    "InventoryChecker",
    variables={
        "sku_id": "1234-XYZ",
        "warehouse_zone": "CN-SH-A"
    }
)
```

子 Agent 内部的推理、工具调用（比如查询数据库）完全在隔离的新 session 中进行，这些中间过程不会回流到主会话。主 Agent 拿到的只是一个干净的 JSON 对象或字符串，可以直接嵌入自己的下一轮回复。

## 踩坑与排障
实际落地时，隔离并非一开了之，有几个让开发者头疼的场景：

**1. 隔离过猛导致子 Agent 失忆**
子 Agent 可能需要知道当前用户身份、时区、权限等级等基础上下文。完全隔离后，这些信息丢失，子 Agent 可能拒绝执行或返回错误。此时必须把必要字段显式加入 `pass_variables`，或者在系统提示词中设计占位符，由主 Agent 动态填充。
```yaml
system_prompt: >
  你是库存助手，当前用户角色：{{ user_role }}，时区：Asia/Shanghai。
```
并在调用时传入 `user_role` 变量。

**2. 工具调用链中的共享状态**
如果子 Agent 内部需要调用某个工具，而该工具依赖主会话中缓存的认证 token 或上下文指针，一旦 session 隔离，这些状态就不可用。解决办法有两种：一是将所需状态通过 `pass_variables` 传递并注入工具调用的参数；二是让子 Agent 自己向认证服务重新获取 token（更安全但增加延迟）。

**3. 并发子 session 的资源风暴**
多个子 Agent 同时被调用，每个都创建独立 session，很容易打满 LLM 提供商的并发限制。需要在上层做并发控制，比如限制同时运行的子 Agent 数量，或使用任务队列串行化部分调用。

**4. 调试跟踪断链**
隔离后，子 Agent 的 trace 默认不会自动关联到主会话的 user_view。排查问题时需要根据返回的 `sub_session_id` 手动搜索。建议在调用处注入一个统一的 `trace_id`，并输出结构化日志：
```python
logger.info("invoking sub_agent", extra={
    "trace_id": trace_id,
    "sub_agent": "InventoryChecker",
    "input_vars": {"sku_id": "..."}
})
```

## 可复用的实践模式
经过几次踩坑，总结出一套在 OpenClaw 项目里可复用的模式：

- **封装调用工厂**：将子 Agent 隔离调用包装为一个函数，预设 `pass_variables` 的必填项，对返回结果做强结构化校验（利用 Pydantic 或 JSON Schema）。这样业务代码里调用时不会遗漏关键变量。
- **动态隔离判断**：不是所有子 Agent 都需要隔离。可以在配置中增加 `isolation_level` 字段（`strict` vs `shared`），根据不同任务类型选择是否继承历史。简单确定性任务用 `strict`，需要跨任务上下文的用 `shared`。
- **记录“隔离收益”**：在测试环境对比同一子 Agent 在隔离与非隔离模式下的成功率、延迟和 token 消耗。用数据说服团队在合适的地方开启隔离，而不是拍脑袋决策。
- **告警脏输出**：对于隔离后仍返回非预期格式的子 Agent，在解析层增加哨兵机制，若输出中包含明显无关字段（如诗歌、代码注释），直接丢弃并触发告警，避免污染主流程。

## 总结
OpenClaw 的 session 隔离并不是一种“设置了就完事”的魔法开关，而是一把需要握好分寸的手术刀。核心原则只有一条：**只给子 Agent 完成当前任务所需的最小上下文**。遵循这一原则，你就能在复杂多任务 agentic 系统中，既保持主会话的连贯性，又让每个子 Agent 像独立微服务一样干净、可控。

工程化的精髓在于把这种隔离做成可配置的管道，而非在每个业务点重复造轮子。当你的系统出现“子 Agent 输出串味”或“主会话被无关细节撑爆”的苗头时，可以回来看看这篇笔记，试试那几行 yaml 和那一个小开关。

---

