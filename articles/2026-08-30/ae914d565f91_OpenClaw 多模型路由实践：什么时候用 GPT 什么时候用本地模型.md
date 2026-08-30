---
title: OpenClaw 多模型路由实践：什么时候用 GPT 什么时候用本地模型
feedId: 35326
source: 综合讨论
publishedAt: 2026-08-30
---

## 背景

在 OpenClaw 上跑 Agent 一段时间后，很容易遇到一个现实问题：不是所有任务都值得调用 GPT 这类云端大模型。结构化提取、意图分类、简单问答用本地 7B/14B 模型就能处理，延迟和成本更低；但复杂推理、长上下文归纳、多步工具编排又必须依赖大模型的能力。如果把所有请求都发给同一个模型，要么成本失控，要么小模型在关键任务上掉链子。

OpenClaw 本身支持多模型后端接入，但默认配置通常只有一个全局模型。要发挥多模型的价值，需要显式地做路由——根据请求特征决定交给哪个模型。这篇文章整理我在 OpenClaw-CN 社区里多次被问到的问题，把可落地的做法写清楚。

## 问题拆解

路由的核心不是“选最好的模型”，而是“用合适的模型处理合适的任务”。具体要回答三个问题：

1. 哪些任务可以安全地下放给本地模型？
2. 哪些任务必须留给 GPT 类云端模型？
3. 路由规则如何配置、如何验证、如何回退？

一个常见误区是“本地模型便宜所以尽量用本地”。但本地模型输出格式不稳定、上下文有限、推理能力弱，如果为了省成本把复杂任务分给它，反而会增加重试和人工介入的成本。反过来，简单任务用 GPT 则纯粹是浪费。

## 做法/步骤

### 1. 定义任务分类

我习惯把 Agent 收到的请求分成四类：

- **确定性操作**：结构化数据提取、实体识别、关键词抽取、格式转换。这类任务输出模式固定，适合本地模型。
- **轻量交互**：FAQ 式问答、状态查询、简单总结。本地模型可以胜任，但需要限定输入长度。
- **复杂推理**：多步骤规划、代码生成、逻辑链分析、长文档综合。这类任务必须用 GPT 类模型。
- **工具编排**：需要调用外部 MCP 工具、生成 tool call 序列、处理多轮工具结果。优先用 GPT 类模型，因为本地模型在 function calling 和工具参数生成上稳定性差。

### 2. 配置 OpenClaw 的模型后端

在 OpenClaw 的配置文件中定义两个模型 provider，例如：

```yaml
model_providers:
  gpt:
    type: openai
    model: gpt-4o-mini   # 或 gpt-4o，按预算选择
    api_key: ${OPENAI_API_KEY}
  local:
    type: ollama
    model: qwen2.5:14b   # 或 llama3.1:8b
    base_url: http://127.0.0.1:11434
```

然后在路由配置中启用规则引擎。OpenClaw 支持基于 `task_type`、`input_length`、`contains_tool_call` 等条件进行分流。

### 3. 写路由规则

下面是一段简化的路由配置示例，逻辑是：先做粗分类，再根据特征进行二次判断。

```yaml
routing:
  rules:
    - name: local_first
      conditions:
        - type: task_type
          operator: in
          value: ["extraction", "classification", "faq"]
        - type: input_length
          operator: lt
          value: 2000
        - type: contains_tool_call
          operator: eq
          value: false
      target: local
      fallback: gpt
    - name: gpt_complex
      conditions:
        - type: task_type
          operator: in
          value: ["reasoning", "planning", "code_generation", "tool_orchestration"]
      target: gpt
      fallback: local
    - name: default
      target: gpt
```

这套规则并不完美，实际运行中需要根据日志调整阈值。比如某些 FAQ 问题虽然输入短，但涉及多轮上下文，本地模型答得不好，就需要把对应分类改成 gpt。

### 4. 加上输出校验和回退

路由不是一锤子买卖。本地模型返回结果后，可以加一层轻量校验：检查输出是否符合预期 schema，或者置信度是否过低。如果校验失败，自动回退到 GPT 重新处理。这样能减少本地模型“一本正经胡说八道”的影响。

在 OpenClaw 中，可以通过 MCP 工具封装一个 `validate_output` 节点，在本地模型返回后调用，失败则走 fallback 链路。

## 踩坑点

1. **本地模型输出格式漂移**：同样是 14B 模型，不同量化版本或不同提示词下，JSON 输出可能突然变成纯文本。建议在 prompt 里强制指定输出格式，并在代码里做严格解析，解析失败直接回退。
2. **上下文窗口被截断**：本地模型上下文通常只有 8k 或 32k，长输入可能会静默截断，导致回答不完整。路由条件里的 `input_length` 要保守设置，不要卡在模型的极限。
3. **function calling 不稳定**：很多本地模型虽然支持 tool call 格式，但参数生成错误率高。不要依赖本地模型做多步工具编排，这类任务优先给 GPT。
4. **影子模式形同虚设**：有人上线路由后不做影子对比，只凭感觉调规则。建议至少跑一周的影子模式，记录同一请求在两个模型下的结果差异，再决定是否切换。
5. **遗漏日志**：路由决策本身要记日志，哪些请求走了本地、哪些回退了、失败原因是什么。没有日志，后续优化全靠猜。

## 可复用建议

- **从保守规则开始**：默认全部走 GPT，只把最确定的任务分流到本地模型。稳定运行一周后再逐步放宽。
- **把路由做成可配置的**：阈值、分类映射、回退开关都放在配置文件里，避免改代码重新部署。
- **用成本指标驱动优化**：记录每个模型的实际 token 消耗和延迟，定期复盘哪些规则真正省了钱，哪些引入了额外的重试成本。
- **关注本地模型的热身和加载延迟**：如果是按需启动的 Ollama 服务，首次调用可能很慢。在路由判断前先做健康检查，避免请求被卡住。
- **不要追求全自动**：初期可以保留人工确认环节，尤其是工具编排类任务。等本地模型表现稳定后再去掉人工介入。

## 总结

OpenClaw 的多模型路由本质上是一个工程决策问题，而不是模型选型问题。GPT 和本地模型不是二选一，而是按任务特征做分流。关键是把路由规则设计成可观测、可回退、可迭代的闭环，而不是一次性配置就完事。

我的实际经验是：约 40% 的请求可以安全分给本地模型，成本下降明显，但前提是做好输出校验和回退。如果你刚开始做，建议先把 FAQ 和结构化提取这两类任务分流出去，跑两周看数据，再决定是否扩展到更多场景。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/31319f7d63f83dc2.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/5e6cc6e12e22d50c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/9aaa134d4da9ab3c.png)

