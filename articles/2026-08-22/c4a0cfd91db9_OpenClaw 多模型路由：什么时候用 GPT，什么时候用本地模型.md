---
title: OpenClaw 多模型路由：什么时候用 GPT，什么时候用本地模型
feedId: 34147
source: 综合讨论
publishedAt: 2026-08-22
---

## 背景

在 OpenClaw 里接入 GPT 之后，很容易把所有任务都丢给云端模型。简单摘要、格式转换、信息提取这些低难度任务用 GPT 也能做，但成本、延迟和隐私并不划算。与此同时，本地模型的能力已经能覆盖不少确定性任务，只是复杂推理、多步规划和 MCP 工具调用仍然容易翻车。混合路由是比较务实的方案：让合适的任务去合适的模型。

## 问题

多模型路由不是简单的关键词分流。实际使用中，主要面临三个问题：

1. **任务边界模糊**：一句“帮我总结并分析这份文档”，既有总结又有分析，关键词规则容易误判。
2. **本地模型能力不稳定**：工具调用参数不准、输出格式不固定、长上下文截断。
3. **回退机制缺失**：本地模型失败后如果没有自动转云端，体验会很差，最后又退回全云端。

所以路由规则需要结合任务类型、上下文长度、工具调用需求和输出格式一起判断。

## 做法/步骤

### 1. 先划分任务池

我通常把任务分成三类：

| 任务类型 | 典型例子 | 推荐模型 |
|---|---|---|
| 纯文本处理 | 摘要、改写、抽取、分类、格式转换 | 本地模型 |
| 结构化/工具调用 | MCP 工具选择、参数填写、多步规划 | GPT |
| 深度推理/长上下文 | 复杂代码生成、长文档分析、多条件推理 | GPT |

划分标准是在本地模型上实际测过 10-20 条用例，确认稳定后再放行，而不是凭感觉。

### 2. 配置双 provider

OpenClaw 的配置示例（不同版本字段可能有差异）：

```yaml
providers:
  cloud:
    type: openai
    api_key: env/OPENAI_API_KEY
    default_model: gpt-4o-mini
    temperature: 0.2
  local:
    type: ollama
    base_url: http://127.0.0.1:11434
    default_model: qwen2.5-coder:14b
    keep_alive: 5m
    temperature: 0.1
```

云端选 `gpt-4o-mini` 做日常复杂任务，本地用 `qwen2.5-coder:14b` 或 `llama3.1:8b` 处理文本工作。如果本地有 GPU，可以上更大的模型；没有 GPU 就用 7b/8b 级别，否则延迟会反超云端。

### 3. 配置路由规则

如果 OpenClaw 版本支持 routing 配置，可以按 `task_type` 分流：

```yaml
routing:
  rules:
    - name: local_text_ops
      match:
        task_type: [summarize, extract, clean, classify]
      model: local
    - name: cloud_reasoning
      match:
        task_type: [planning, coding, multi_tool, long_context]
      model: cloud
  fallback: cloud
```

如果当前版本不支持图形化路由，可以拆成两个 executor：一个 `cloud-executor`，一个 `local-executor`。主控 agent 只做意图分类，返回 JSON：

```json
{"task_type": "extract", "model": "local"}
```

然后根据 JSON 调用对应 executor。主控 agent 本身用轻量本地模型即可，分类任务足够。

### 4. 处理长上下文和工具调用

本地模型上下文通常较短。在路由前增加 token 长度判断：输入超过 12k token 或预计输出较长时，直接转 GPT。工具调用同理：如果任务涉及多个 MCP 工具或复杂参数，本地模型很容易生成错误参数，优先交给云端。

## 踩坑点

- **本地模型 JSON 输出不稳定**：即使 prompt 要求 JSON，也可能多输出前后缀。建议启用 constrained decoding 或做一层后处理，但重试次数限制在 2 次以内，否则本地耗时反而超过云端。
- **关键词路由误判**：不要用“总结”“分析”这类词做硬规则。用主控 agent 做一次意图分类，或者至少加一层否定条件。
- **本地模型工具调用弱**：不是所有本地模型都擅长 function calling。`qwen2.5` 系列对工具调用支持相对好，但多参数工具仍建议云端规划。
- **延迟错觉**：CPU 跑 14b 模型时，简单任务可能比 GPT-4o-mini 还慢。先测 p50/p95 延迟，p95 超过 10 秒的任务要谨慎走本地。

## 可复用建议

1. **维护模型能力清单**：记录每个模型的上下文长度、工具调用能力、输出格式稳定性、典型延迟。每换模型跑一遍固定测试集。
2. **设置回退链**：本地失败 → 自动转 GPT，但记录失败原因。失败率超过 15% 的任务类型，暂时不要走本地。
3. **用 token 长度和温度辅助判断**：短输入、低温度用本地；长输入、需要多样性的任务用云端。
4. **先跑通“云端规划 + 本地执行”**：不要一开始就让本地模型参与复杂决策，先从纯文本处理切入，稳定后再扩展。
5. **控制本地模型可用工具数量**：本地 executor 只保留无参数或单参数 MCP 工具，减少出错面。

## 总结

多模型路由的核心是让能力、成本、延迟匹配任务需求，不是追求“全本地替代云端”。在 OpenClaw 里，更稳妥的路径是：GPT 做判断、规划、复杂工具调用；本地模型做清洗、提取、分类、短文本处理。先跑通一个混合配置，再逐步扩大本地模型覆盖范围，比一次性替换要可靠得多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/76a30a768484b68a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/83f48dba12801b60.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/babcfdc78abc41b4.png)

