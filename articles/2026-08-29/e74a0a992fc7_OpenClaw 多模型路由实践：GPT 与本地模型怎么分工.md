---
title: OpenClaw 多模型路由实践：GPT 与本地模型怎么分工
feedId: 35144
source: 综合讨论
publishedAt: 2026-08-29
---

## 背景

OpenClaw 里接多个模型越来越常见：一边是 GPT 这类云端模型，能力强、工具调用稳定，但按量付费、有延迟、数据要出网；另一边是本地模型（Qwen2.5、Llama3.1、DeepSeek 等）通过 Ollama/vLLM 暴露接口，几乎零 token 成本、数据留在本机，但复杂推理和工具调用不如云端。

问题不是选哪个，而是怎么让请求自动走到合适的模型。

## 问题

如果不做路由，会出现两个极端：要么所有请求都走 GPT，月度 token 账单高，简单任务也要等网络；要么一刀切走本地模型，遇到复杂任务时本地模型开始乱编工具参数、忘记约束，甚至把 agent 流程带崩。

多模型路由的目标是：高频低复杂度任务本地消化，高价值复杂任务交给 GPT，同时保留失败回退。

## 做法/步骤

### 1. 先梳理任务类型

把 OpenClaw 里的常见请求分类：

- 意图识别、实体抽取、摘要、改写、格式转换、分类、简单问答：优先本地；
- 代码生成、复杂推理、多步工具调用、长文档分析、强指令遵循：优先 GPT。

### 2. 注册模型并配置路由层

在 OpenClaw 里分别注册本地 provider 和 GPT provider。不要在业务脚本里 if-else 判断模型名，而是加一个 router/middleware。我一般用一个 YAML 配置描述规则：

```yaml
router:
  default: local
  rules:
    - if:
        tool_calls: true
      then: gpt
    - if:
        context_tokens: "> 4000"
      then: gpt
    - if:
        data_sensitive: true
      then: local
    - if:
        task_type: [summarize, classify, extract, rewrite]
      then: local
    - if:
        intent: [code_generation, complex_reasoning, multi_step_plan]
      then: gpt
  fallback:
    local_fail: gpt
    gpt_fail: local
```

这是简化示例，实际可以根据 prompt 长度、是否包含工具调用、是否包含敏感字段、任务标签来组合。

### 3. 本地模型做能力边界控制

本地模型不要直接处理长上下文和复杂工具。用 Ollama 加载 7B/14B 级别模型，设置 `temperature=0` 或很低，打开 JSON schema 约束，限制 `num_ctx` 不要超过 4096 或 8192。请求进入本地模型前先截断或分片，但分片要谨慎，避免摘要质量下降。

### 4. 输出校验与自动回退

本地模型返回后做格式校验，例如 JSON 解析、字段完整性、长度合理性；校验失败就回退 GPT。GPT 超时或额度不足时，降级到本地模型并标记 degraded。

### 5. 记录路由日志

每条请求记录 `route_decision`、`model`、`tokens`、`latency`、`cost`、`fallback` 字段，方便后续复盘。

## 踩坑点

- **拿本地模型硬扛 agent 工具调用**：7B 模型经常把工具名写错、参数类型乱给，或者漏掉必填字段。解决办法是工具调用场景强制走 GPT，或者本地模型只做无工具子任务。
- **规则只看关键词**：比如检测到“代码”就转 GPT，但用户可能只是问“代码里这个报错是什么意思”，本地模型完全能答。需要结合上下文长度、是否真的是生成任务。
- **长文档无脑切块**：本地模型 context 小，切块后摘要会丢失跨块信息。长文档直接走 GPT，或者先提取每个块的关键点，再本地汇总，但效果有限。
- **忽略输出契约**：本地模型输出非 JSON 导致下游插件失败，回退机制没生效。一定要用 JSON schema 约束，并做解析失败计数。
- **本地模型并发打爆显存**：多个 OpenClaw 任务同时进本地模型，Ollama 排队导致超时。需要限制并发或使用队列，设置合理超时。

## 可复用建议

- **先影子模式运行**：路由规则先不真正切换模型，只记录建议路由结果，和实际使用对比，跑一周再调整。
- **用成本与延迟指标校准**：统计本地命中率、回退率、平均每请求成本、P95 延迟。如果本地回退率超过 15%，说明规则或模型能力有问题。
- **敏感数据优先本地**：涉及内部文档、密钥、隐私信息，直接本地模型处理，除非任务复杂度确实无法支撑。
- **保留一键全局切换**：OpenClaw 侧保留强制全部走 GPT 或全部走本地的开关，用于额度紧张或本地服务故障时快速切换。

## 总结

多模型路由不是“本地替代 GPT”，而是让 GPT 只出现在它真正值得出现的地方。我目前的经验是：约 60%-70% 的简单任务可以本地消化，剩下的复杂、工具调用、长上下文任务再走 GPT。这样既保证 agent 稳定，又能明显降低费用和响应时间。关键是路由规则要可观测、可回退，别让省钱的策略变成排障的负担。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/3e40244e861994c3.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/a3f0824baaf1ffde.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/0b4e8f8f6fcc49ab.png)

