---
title: OpenClaw 多模型路由实践：GPT 与本地模型的分工边界
feedId: 35011
source: 综合讨论
publishedAt: 2026-08-28
---

最近把 OpenClaw 从单模型切到多模型路由，起因是两周内 API 成本涨了不少，但很多请求其实不需要 GPT 级别能力。另一个因素是某些自动化任务包含内部日志和密钥片段，不想送到外部 API。本文记录当前使用的路由策略和踩坑。

## 背景

OpenClaw 作为 Agent 运行时会同时处理多种任务：MCP 工具编排、插件调用、对话总结、定时任务、文件解析等。这些任务对模型能力、延迟、隐私要求不同。全部用 GPT 会产生不必要的成本和隐私暴露；全部用本地模型，复杂工具调用和长上下文表现不稳定。多模型路由的目标是让每类任务进入合适的 provider，而不是简单地二选一。

## 问题

我的初始需求有三类：

1. 涉及 MCP 工具调用、复杂 JSON 参数的任务，能力优先。
2. 涉及内部路径、日志、敏感配置的文本处理，隐私优先。
3. 高频小任务如状态分类、摘要、句式改写，成本优先。

如果手动切换模型，每次要改配置并重启，而且容易遗忘。需要一种规则化的自动路由。

## 做法/步骤

### 1. 配置两个 provider

本地使用 Ollama 挂 Qwen2.5 14B 或 Llama 3.1 8B，云端使用 GPT-4o mini 或 GPT-4o。OpenClaw 中分别配置 provider，建议给本地模型设置较长的请求超时，例如 120 秒，因为 CPU 推理可能慢。

### 2. 定义路由规则

我在 OpenClaw 的模型路由配置中为不同任务打标签，再用规则匹配。以下是一个简化版：

```yaml
routing:
  default: local
  rules:
    - when:
        task_type: [mcp_tool_call, complex_code]
      model: gpt
    - when:
        sensitive_data: true
      model: local
      forbid_fallback: true
    - when:
        task_type: [summary, classify, rewrite]
      model: local
    - when:
        context_tokens: ">6000"
      model: gpt
```

实际字段名以你的 OpenClaw 版本为准。核心规则是：默认本地，工具调用和长上下文升级云端，敏感数据锁定本地。

### 3. 为 MCP 工具加约束

本地模型在生成工具调用 JSON 时容易多一个尾逗号或漏字段。我会在 MCP 工具的 description 里写清参数 schema，并尽量让工具名简短。部分平台可用 grammar/JSON schema 约束，优先开启。

### 4. 设置 fallback 与日志

每个非敏感规则保留 fallback：本地解析失败、超时、工具调用格式错误时自动切到云端。日志中记录 `routing_reason`，方便回看为什么选择某个模型。

## 踩坑点

- **本地小模型工具调用不稳定**：7B/8B 模型能聊天但工具调用 JSON 错误率较高。我的经验是 14B 以上且开启结构化输出才适合做简单工具调用。MCP 复杂编排仍建议 GPT。
- **长上下文被本地模型截断**：自动化任务常把日志塞进上下文，本地模型上下文窗口小，容易漏掉中间内容。路由条件里要加 `context_tokens` 判断，超过阈值直接进云端。
- **隐私 fallback 的坑**：最初我给所有规则都配了云端 fallback，结果敏感任务本地失败后自动发送到云端。后来加了 `forbid_fallback: true` 或独立规则，确保敏感任务不会因为“失败自动升级”而外泄。
- **切换不彻底**：修改路由配置后 OpenClaw 可能缓存旧模型列表，部分插件进程仍持有旧连接。建议重启 OpenClaw 和相关 MCP 子进程，而不仅是 reload。
- **CPU 推理延迟**：本地模型在无 GPU 的机器上可能比 API 还慢，不适合需要 2 秒内响应的交互式 Agent。如果延迟敏感且无 GPU，降低本地模型尺寸，或改为关闭本地路由。

## 可复用建议

1. 建立一张任务分类表：能力、隐私、延迟、成本四个维度，每个任务标注优先级。
2. 路由策略优先采用“默认本地，升级云端”，比“默认云端，降级本地”更可控。
3. MCP 复杂工具调用不要交给小模型，除非你明确知道模型的结构化输出稳定。
4. 给所有路由规则加 `routing_reason` 日志，至少观察一周再调整。
5. 对敏感数据使用 `local-only` 或 `forbid_fallback`，不要用默认的自动升级。
6. 定期抽查本地模型输出质量，特别是有格式约束的任务，避免“能跑但数据脏”。

## 总结

OpenClaw 的多模型路由不是简单省钱，而是把不同的任务分配合适的模型能力。我的当前策略是：本地模型处理隐私、摘要、分类和简单改写，GPT 处理 MCP 工具编排、长上下文和复杂生成。规则稳定运行后，API 成本下降约三分之一，同时敏感日志不再出本机。这个方案不会一步到位，需要根据错误日志和成本数据逐步调整。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/b1a51e9344fade4d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/565461dec2072dc5.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/2b2f2dc7800fdedc.png)

