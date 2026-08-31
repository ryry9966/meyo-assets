---
title: OpenClaw 的 session 隔离：让子 Agent 只交结果，不交草稿
feedId: 35621
source: 综合讨论
publishedAt: 2026-09-01
---

## 背景

在 OpenClaw 上做多 Agent 编排时，常见模式是：主 Agent 负责拆解目标、选择工具、合并结果；子 Agent 负责执行搜索、文件处理、代码生成等具体子任务。子 Agent 经常需要多轮尝试：访问 MCP 工具、读文件、遇到错误再调整策略。如果不加约束，这些过程会直接追加到主 Agent 的 session，主上下文很快被中间过程淹没。

## 问题

不加隔离时，子 Agent 至少会带来三类污染：

1. **过程噪声**：工具的原始输出、失败重试、中间推理全部进入主会话。
2. **状态误读**：主 Agent 可能把子任务的中间错误当成最终状态，影响后续计划。
3. **成本与延迟**：上下文越长，模型推理越慢，token 成本越高，而且容易注意力漂移。

很多用户一开始只是为了“省事”，让子 Agent 直接在主 session 里跑，结果几轮下来主 Agent 开始重复询问已处理过的问题，或者把子 Agent 的失败日志当成事实。

## 做法 / 步骤

OpenClaw 支持子 Agent 的 session 隔离。核心思路是：主 session 只把子任务“派发”出去，子 Agent 在独立 session 中运行，只返回最终结构化结果。

**1. 定义子 Agent 的输入输出契约**  
例如统一返回 JSON：`{status, summary, data, error}`，其中 `summary` 只保留主 Agent 需要知道的结论，`data` 只放必要数据。

**2. 在 spawn 子 Agent 时启用隔离**  
OpenClaw 配置中可以使用类似设置：

```yaml
subagent:
  isolation: full
  context_return: result_only
  max_steps: 12
  output_schema: subagent_result
```

代码侧可以这样写：

```python
sub = session.spawn(
    task_payload,
    isolation="full",
    return_mode="final_output",
    max_steps=12,
    result_schema=SubResultSchema,
)
result = sub.run()
main_session.add_observation({
    "summary": result.summary,
    "data": result.data
})
```

**3. 子 Agent 内部继续使用 MCP 工具和插件**  
过程消息只保留在子 session 中，主 session 只接收 `add_observation` 写入的精简结果。子 Agent 可以自由试错，不会反向污染主会话。

**4. 并行子 Agent 时用 parent_id 关联**  
每个子 Agent 创建独立 session，并在日志系统中记录 parent_id，方便追踪和成本聚合。

## 踩坑点

- **没有约束输出格式**：子 Agent 的 `final_output` 可能仍然包含大段推理过程。务必加 schema 校验和 `max_output_tokens`。
- **隔离后上下文丢失**：子 Agent 看不到主 session 的历史，可能会重复提问。派发任务时要把必要背景打包进 `task_payload`，不要假设子 Agent 能“记住”主会话。
- **子 session 悬挂**：主任务取消或超时后，子 session 可能仍在运行。设置 TTL 和取消传播，避免资源泄漏。
- **共享状态冲突**：多个子 Agent 同时写同一文件或数据库，容易竞争。尽量让子 Agent 使用独立工作目录，结果通过返回对象传递，共享数据通过只读 MCP 暴露。
- **调试不可见**：子 session 内部过程看不到，排查困难。建议开启 trace 保留子 session 日志，但不要把这些日志回灌主 session。

## 可复用建议

- 统一子 Agent 返回 envelope：任何子 Agent 都返回 `status / summary / data / error` 四个字段。
- 所有子 Agent 显式设置 `max_steps`、`max_output_tokens` 和超时时间。
- 主 session 只做编排和决策，子 session 做执行和探索；不要让主 Agent 直接阅读子 Agent 的完整工具日志。
- 在日志和成本系统中用 `parent_id` 聚合子 session 的消耗，观察隔离效果。
- 对只读数据，优先通过 MCP 工具提供，而不是让子 Agent 直接访问共享文件。

## 总结

session 隔离不是简单开启一个开关，而是一套配合结构化输入输出、生命周期管理和调试策略的工程实践。做好隔离后，子 Agent 的试错过程不会再污染主会话，主 Agent 的决策上下文保持干净，整体编排也会更稳定、成本更可控。对 OpenClaw 的多 Agent 场景来说，这应该是默认配置，而不是出了问题才补的补丁。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/12091c5204b594c5.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/b5ba328d9f7c4f75.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/8ef9fc6184aa4f71.png)

