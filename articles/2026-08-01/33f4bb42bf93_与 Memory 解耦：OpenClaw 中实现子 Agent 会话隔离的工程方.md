---
title: 与 Memory 解耦：OpenClaw 中实现子 Agent 会话隔离的工程方法
feedId: 31241
source: 综合讨论
publishedAt: 2026-08-01
---

## 问题背景

在 OpenClaw 的多 Agent 协作场景中，主 Agent 通常会借助 `delegate_task` 或 `sub_agent_call` 等机制将子任务派发给专门的子 Agent。初期实现很直接：把子 Agent 的完整推理‑行动循环（包括 thinking、tool call、observation）全部追加到当前 session 的对话历史中。这样做刚开始“跑得通”，但当系统进入生产阶段后，几个副作用逐渐暴露：

1. **上下文窗口快速膨胀**：主 Agent 的 session 会迅速被子任务的交互细节占满，导致其对后续主任务的理解能力下降，甚至需要人工截断历史。
2. **提示词污染与越权风险**：子 Agent 在某些场景下接收到的指令、中间密钥、内部数据，会直接残留在主 session 里，后续如果再次使用主 session 进行总结、导出或审计，可能造成敏感信息泄露。
3. **调试与回溯混乱**：出问题时，一眼看过去全是子 Agent 的中间输出，难以定位主 Agent 自身的决策路径。

因此，**session 隔离** 不是“锦上添花”，而是多 Agent 系统从演示级走向工程级的临界优化。

## 核心思路：解耦 conversation，只传递结构化结果

OpenClaw 的 session 本质上是一段 `messages` 列表（类似于 OpenAI message 格式），并绑定到唯一的 `session_id`。传统做法是让主 Agent 和子 Agent 共享同一个 `session_id`，对话自然会被 merge。反过来，只要做到两点，就可以隔离：

- 给子 Agent 分配**独立的 session**（或使用 `parent_id` 派生出的隔离层）。
- 子 Agent 执行结束后，仅将**结构化的、最少必要的结果**合并回主 Agent 的 session，而丢弃其内部的思考链和工具调用细节。

这类似于 Unix 进程的 fork/join 模型，子进程不能污染父进程的内存空间。

## 工程实现步骤（基于 OpenClaw 0.3+）

下面是在一个内部文档问答机器人中实际可行的实现路径，供参考。

### 1. 定义子 Agent 时强制指定隔离策略

在 OpenClaw 的配置文件中，给每个子 Agent 增加 `session_isolation: strict` 标记。如果框架原生不支持，则通过插件或中间件自行实现。核心是拦截 `run_agent` 调用：

```python
async def isolated_sub_agent_run(agent_name, task, parent_session_id):
    sub_session_id = f"{parent_session_id}--sub:{agent_name}:{uuid4().hex[:8]}"
    # 创建全新的 session
    session = await session_store.create(sub_session_id, system_message=agent.system_prompt)
    # 注入任务消息
    await session.add_user_message(task)
    # 运行子 Agent
    result = await agent.run(session)
    # 仅提取最终结构化输出
    final_output = extract_structured_output(result)
    # 销毁子 session（或标记为只读存档）
    await session_store.delete(sub_session_id)
    return final_output
```

### 2. 主 Agent 的工具函数返回“摘要”而非原始消息

在主 Agent 的工具定义（例如 `use_sub_agent`）中，返回的 `content` 不要直接拼回 `assistant` 的原始消息，而是一段简洁的 JSON 摘要：

```json
{
  "sub_agent": "sql_analyzer",
  "task": "统计上周异常告警",
  "result": {
    "count": 12,
    "top_category": "db_timeout",
    "suggest_action": "扩容连接池"
  },
  "cost_tokens": 450,
  "duration_s": 2.3
}
```

同时，在工具描述中明确说明：“不要将子 Agent 的对话历史作为上下文输出，仅输出此 JSON 结构。”这样可以训练主 Agent 在 prompt 中遵守该契约。

### 3. 清理 main session 中的残留引用

即使工具返回了结构化数据，也要防止主 Agent 无意中把之前子调用的中间输出再进行联想。可以在主 Agent 的 construction config 中加入：

```yaml
memory:
  max_message_window: 20   # 限制主 session 保留的消息数量
  exclude_tool_call_details: true   # 剔除工具调用的参数细节
```

另外，在每次主任务结束时，调用 `session.compact()` 方法，根据重要性压缩历史，丢弃子 Agent 引入的 noise。

## 踩坑记录

1. **子 Agent 的上下文不足**  
   如果完全隔离，子 Agent 就失去了主历史中的关键背景。解决办法是允许主 Agent 在调用时传入一个 `context_preview` 字段（如最近 3 轮主对话的摘要），但依然保持 session 实体隔离。

2. **嵌套子 Agent 的 session 命名冲突**  
   当子 Agent 再调用次子 Agent 时，session_id 可能过长或重复。实践上采用三段式命名：`{root_session}::{level1_agent}::{level2_agent}`，并通过 `session_lifespan` 属性自动 TTL 回收。

3. **审核需求方的反对**  
   安全团队最初要求保留全量日志用于合规。最终方案是：子 session 归档为只读日志，主 session 保持清洁；事故排查时通过 `parent_id` 追溯完整调用链。

4. **MCP 工具上下文泄露**  
   如果子 Agent 通过 MCP 调用了外部工具，这些工具的响应也会残留在子 session 中。如果忘记销毁子 session，敏感数据会存活在内存。务必在 `finally` 块中确保删除。

## 可复用的工程建议

- **统一结构化输出规范**：为所有子 Agent 约定一个 `SubAgentResult` schema，强制使用，避免随意返回。
- **建立 session 生命周期策略**：`isolated` 类型的子 session 默认存活时间 5 分钟，到期自动清除；`audit` 类型则归档到冷存储。
- **在监控中加入 session 脏度指标**：统计主 session 中来自子 Agent 消息的 token 占比，超过阈值自动告警。
- **利用 OpenClaw 的 Hook 机制**：在 `on_sub_agent_finish` 钩子中自动执行清理与格式校验，避免开发者遗忘。

## 总结

子 Agent 不污染主会话，本质上是对**上下文空间的分级管理**。简单的多 Agent 几乎不需要它，但一旦你的 OpenClaw 应用需要长时间稳定运行、或者要面对安全审计，隔离就是必选项。实现方法并不复杂：独立 session + 结构化返回 + 自动清理。把它作为一个固定的工程脚手架应用到每个子 Agent 上，后续的维护成本会显著降低。

---

