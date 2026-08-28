---
title: OpenClaw 多模型路由实践：GPT 与本地模型各管一段
feedId: 35071
source: 综合讨论
publishedAt: 2026-08-28
---

## 背景
在 OpenClaw 的 Agent、MCP 和插件链路里，模型调用点越来越多。单一用云端 GPT，成本和延迟会随调用量线性上升；全部切到本地模型，又会在复杂推理、长文档和工具调用上频繁失败。多模型路由不是“为了省钱把请求硬塞给本地模型”，而是让不同任务落到合适的模型上。

## 问题
需要判断的维度至少包括：复杂度、上下文长度、失败代价、数据敏感度、结构化输出要求、是否需要稳定 function calling。如果只看“便不便宜”，很容易把关键任务打到本地模型上，最后花更多时间重试和人工兜底。

## 做法/步骤
1. 先列举 OpenClaw 里的模型调用点：MCP 工具返回内容摘要、日志/消息分类、意图识别、实体抽取、敏感信息脱敏、代码生成、长文档问答、多步规划。
2. 定义维度：建议把任务分成两类。一类是“高重复、低失败代价、可校验”，一类是“低重复、高失败代价、强推理”。
3. 接入双通道：本地模型走 Ollama 或 vLLM，先上 7B-14B instruct 模型；云端走 GPT 或同类强模型。
4. 配置规则路由。逻辑示例：

```yaml
routes:
  - name: local_first
    match:
      task_type: [classification, extraction, redaction, summarization]
      max_input_tokens: 3000
      allow_retry: true
    models:
      primary: local_qwen_14b
      fallback: gpt-4o-mini

  - name: cloud_first
    match:
      task_type: [planning, code_agent, long_context_qa, tool_planning]
      max_input_tokens: 32000
      need_function_calling: true
    models:
      primary: gpt-4o
      fallback: local_qwen_14b
```

这是规则逻辑，不是 OpenClaw 原生语法，可以写成 router 插件或配置文件。

5. 回退与熔断：本地模型超时、JSON 不合法或输出重复时切到 GPT；GPT 限流或超时切本地，但只允许处理低风险任务。熔断要有冷却时间，避免抖动。
6. 日志与评估：记录命中哪条路由、模型、耗时、token、是否回退、结果是否可用。稳定运行一周再调整阈值。

## 踩坑点
- 本地模型的 function calling 不稳定，尤其是低量化版本，容易漏参数或编造字段。不要让它直接生成复杂 MCP 调用参数。
- 上下文长度虚标：部分推理框架会截断而不是报错，导致中间信息丢失。长输入任务要显式限制，并优先走 GPT。
- 提示词不通用：GPT 能遵循的长 system prompt，本地模型可能跑偏。建议为本地模型单独压缩提示词，减少角色扮演和长指令。
- 本地推理并发低：Agent 多步骤循环容易排队超时。把本地模型限制在单轮、批处理类任务。
- 规则路由漏判：枚举任务类型不全，长尾请求会走默认云端，成本没降下来。每周查日志补规则。

## 可复用建议
- 先用规则路由，等日志足够再考虑语义路由或小模型打分，不要一上来做复杂路由。
- 本地模型适合分类、抽取、脱敏、摘要、格式转换；GPT 适合多步规划、代码生成、长上下文推理和稳定工具调用。
- 把 GPT-4o-mini 作为本地模型的回退，而不是主模型，能兼顾成本与稳定性。
- 为本地模型输出加 schema 校验，失败自动回退比反复重试更省时间。
- 设置预算：例如本地单任务超过 3 秒或两次重试失败，就切云端。

## 总结
多模型路由是工程折中。合理的分工是：本地模型承担高重复、低风险、可验证的工作，GPT 承担高复杂度、长上下文和关键路径。前期用规则路由跑起来，用日志校准，再逐步自动化，比盲目切换模型更靠谱。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/0d6a05c369f5f3dc.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/9df17bf5b9c81874.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/6e2d65de89eb4f79.png)

