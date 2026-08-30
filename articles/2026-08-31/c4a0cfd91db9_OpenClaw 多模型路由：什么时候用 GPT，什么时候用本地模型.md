---
title: OpenClaw 多模型路由：什么时候用 GPT，什么时候用本地模型
feedId: 35468
source: 综合讨论
publishedAt: 2026-08-31
---

## 背景

OpenClaw 里 Agent 经常面对两类任务：一类是高频、低难度、数据敏感的管道任务，比如 MCP 返回清洗、字段抽取、摘要；另一类是低频、高难度、需要规划或复杂工具调用的任务。全量用 GPT，成本、延迟、数据外发都会随请求量线性上涨；全量用本地模型，复杂推理、长上下文、函数调用会很快变成瓶颈。多模型路由就是按任务特征把请求分给合适的模型。

## 问题

常见失败模式有两个：

- **全量 GPT**：让 GPT 对 30KB 的 MCP 返回做字段提取，实际只是模式匹配，浪费 token。
- **全量本地**：让 7B 模型做多步 Agent 规划，常出现工具名幻觉、参数缺失、JSON 破损，导致执行链中断。

关键不是“哪个模型更好”，而是“这个任务值不值得用 GPT”。简单判断：需要理解上下文或多步推理的给 GPT，只做模式识别的给本地模型。

## 做法/步骤

1. **给任务打标签**：在 OpenClaw 的 task metadata 里标记输入长度、任务类型、是否需工具调用、隐私级别、是否允许重试。路由之前先回答三个问题：输入多长？输出能不能校验？失败能不能重试？
2. **建立路由规则**：本地模型适合抽取、摘要、分类、MCP 结果整理；GPT 适合规划、工具选择、代码生成、长上下文推理。如果任务只需从固定结构里取字段，本地模型通常够用；如果涉及多步推理或模糊指令，必须 GPT。默认 GPT 小模型兜底，本地模型前置处理确定性任务。
3. **配置回退与观测**：本地模型输出先做 JSON schema 校验，失败记录 `fallback_reason` 并切 GPT；同时记录模型、耗时、token、失败原因，每周回看命中率。

示意配置：

```yaml
routing:
  default: gpt-4.1-mini
  rules:
    - when: {task_type: [extraction, summarization], requires_tool_calls: false}
      model: local_qwen2.5_7b
      fallback: gpt-4.1-mini
    - when: {task_type: [planning, tool_use], requires_tool_calls: true}
      model: gpt-4.1
```

## 踩坑点

- 本地模型 function calling 输出字段不稳定，必须做 schema 校验，不要裸接 tool_use。
- 长上下文工具结果直接丢给本地模型容易被截断，先 chunk 或只处理关键字段。
- 路由规则过细会让维护成本高于省下的 token，先覆盖 3-5 个高频场景。
- 忘记设置本地模型超时，Agent 会卡住；必须超时回退。
- 路由判断不要用模型自己决定；用规则判断更省 token、更可控。
- 本地模型在不同 prompt 语言混合下输出格式容易漂移，固定输出示例比长提示更有效。

## 可复用建议

- 默认 GPT 小模型兜底，本地模型只前置处理确定性任务。
- 把 MCP 结果整理作为本地模型的第一站，收益最明显。
- 所有 `fallback_reason` 必须落日志，这是调整规则的唯一依据。
- 每周重放失败样本，验证新本地模型是否可以接管。
- 路由先覆盖 3-5 个高频场景，稳定后再扩展，不要一次写几十条规则。

## 总结

多模型路由不是选最强模型，而是让便宜模型吃掉高频低难度流量，让 GPT 处理关键路径。先做少量清晰规则，配好回退和日志，再根据失败原因渐进调整，比一次性上复杂路由更稳定。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/ff082fe61a0f72ff.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/0e759f6ec14e6aad.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/709fbe5777a35b4e.png)

