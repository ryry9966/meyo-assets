---
title: OpenClaw 多模型路由实战：什么时候用 GPT，什么时候用本地模型
feedId: 35741
source: 综合讨论
publishedAt: 2026-09-02
---

## 背景

Agent 跑起来之后，模型调用是最大的成本和风险变量。我们社区里不少人的现状是两个极端：要么全走 GPT 级云端模型，一天账单几十上百，批量摘要、格式清洗这种活也在烧钱；要么全压在本地 7B/14B 上，结果 planning 和多工具编排时 tool call 频繁格式崩坏。任务本身不是同质的，路由才有意义。

## 问题

路由决策缺一个明确的分层标准。大家常问的其实是三个问题：哪些步骤值得用强模型、哪些数据不能出内网、本地模型的能力边界到底在哪。没有这套判断，路由规则就是拍脑袋写的。

## 做法

**1. 先按三个维度给任务分层**：是否需要可靠的 tool calling、上下文长度、是否涉及敏感数据。实践中比较稳的分法：

- S 级：任务规划、多 MCP 工具编排、代码生成 → 云端强模型
- A 级：单工具调用、长文摘要 → 本地中档模型或云端小模型
- B 级：格式化、批量分类、预处理 → 本地小模型

**2. 在 OpenClaw 的 model registry 里注册模型并打能力标签**，路由规则按 step tag 匹配，并配置 fallback 链：

```yaml
models:
  gpt-high:  { provider: openai, model: gpt-4.1, tags: [tool-call, long-ctx] }
  local-14b: { provider: vllm, model: qwen2.5-14b-instruct, tags: [tool-call, private] }
  local-7b:  { provider: ollama, model: qwen2.5:7b, tags: [private, cheap] }

routes:
  - match: { step: planning }
    use: gpt-high
    fallback: [local-14b]
  - match: { step: summarize, data: sensitive }
    use: local-14b
  - match: { step: format }
    use: local-7b
```

**3. 路由做在 step 级，不要做在 agent 级。** 同一个 agent 里，规划步骤和格式化步骤的能力需求差得很远，agent 级路由粒度太粗。

**4. 每条路由加日志**：命中规则、实际模型、token 数、耗时、是否触发 fallback。没有这些字段，后面没法调。

## 踩坑点

1. **“声称支持 tool call” ≠ 稳定。** 14B 级别尚可，7B 在参数多或 schema 嵌套时经常输出坏 JSON。tool-call 门槛要以实测为准，别信 model card。
2. **长上下文静默截断。** 本地 8k 窗口的模型接到长 MCP 返回时不会报错，只会悄悄丢内容，表现为“答非所问”。路由前先估 token，超限直接走长上下文模型。
3. **fallback 风暴。** 云端超时 → 本地接手 → 输出不合格 → 重试，双倍成本。给 fallback 设次数上限，降级路径要在输出里标注。
4. **embedding 别参与路由。** 切 embedding 模型等于重建向量库，单独固定一个，永不切换。
5. **别用关键词启发式路由。** “prompt 里有没有‘总结’两个字”这种规则极脆，用显式 step tag 声明。

## 可复用建议

- 建 20~50 条评测集（tool call、长上下文、敏感数据各覆盖几条），每次改路由规则先跑一遍看命中率。
- 监控两个指标：各档模型调用占比、fallback 率。fallback 率持续超过 10%，说明分层或模型选型有问题。
- 给路由加预算上限，配额耗尽自动降档，避免月底账单惊喜。
- 敏感数据由调用方显式声明 `data: sensitive`，不要指望运行时自动识别。

## 总结

多模型路由的价值不只是省钱，而是把不同性质的任务交给成本匹配的模型，同时让敏感数据留在内网。从 step 级路由、实测 tool-call 门槛、fallback 限流这三件事做起，一两天就能落地，之后靠评测集和监控持续调优即可。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-02/5564555fcad3bdd3.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-02/66e0c55b89b34829.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-02/a90299778a4a2df8.png)

