---
title: OpenClaw 多模型路由：GPT 与本地模型的分工边界
feedId: 34814
source: 综合讨论
publishedAt: 2026-08-26
---

## 背景

在 OpenClaw 的 Agent 链路里，模型不再只是“聊天补全”，而是会被 MCP 工具、插件和自动化任务反复调用。单用 GPT 时，高频、低难度任务会带来不必要的 token 成本、延迟和隐私暴露；单用本地模型时，复杂推理、长上下文和工具选择又容易翻车。多模型路由解决的其实不是“哪个模型更强”，而是怎么把不同成本、能力、风险的请求分到合适的执行单元。

## 问题

真正影响路由判断的通常是三类特征：

- 任务复杂度：是否需要多步规划、长上下文推理、复杂代码调试。
- 数据敏感度：是否包含内部数据、密钥、PII 或不适合出域的内容。
- 失败成本：一次失败是重试一下就好，还是会中断自动化流程或污染下游数据。

简单说，GPT 适合做“决策和工具协调”，本地模型适合做“高频、低难度、隐私敏感的脏活”。

## 做法/步骤

我目前的做法是先用静态规则，再逐步加入轻量分类。大致路由表如下：

```yaml
routes:
  - name: privacy_sensitive
    match:
      contains_pii: true
    backend: local
  - name: multi_tool_plan
    match:
      required_tools: true
      max_tool_steps: ">3"
    backend: remote
  - name: simple_extract
    match:
      intent: [extract, summarize, classify, rewrite]
      context_tokens: "<8000"
    backend: local
    fallback: remote
  - name: long_context_qa
    match:
      context_tokens: ">=8000"
    backend: remote
```

实际配置字段可能随 OpenClaw 版本略有差异，但核心是：先按数据敏感度强制分流，再按任务类型和上下文长度选择本地或远程。

本地模型侧，我一般用 7B/14B 的量化版本，例如 Qwen2.5-7B-Instruct 或同级别的可商用模型。关键不是模型有多大，而是给它受控输出：

- 开启 JSON schema 或 grammar 约束；
- 温度设到 0.1-0.3；
- 限制 max_tokens；
- 只让本地模型做抽取、摘要、分类、改写，不做复杂工具调用。

回退逻辑也必须在网关层做，而不是在业务代码里到处 try/catch。伪代码可以简化成：

```python
if route.backend == "local":
    try:
        result = local.run(task, json_mode=True)
        validate_schema(result)
        return result
    except LocalModelError:
        return remote.run(task)
return remote.run(task)
```

远程模型同样需要降级路径。比如 GPT 限流或网络不可用时，可以降级为本地模型的只读任务，但禁止进入多步工具调用。

## 踩坑点

1. 本地模型在 MCP 工具超过 5 个时，经常选错工具或填错必填参数。7B 模型不是不能调工具，而是不适合“大海捞针式”工具选择。如果必须本地工具调用，就把候选工具收到 1-3 个，或使用专门的 function-calling 微调模型。
2. 依赖 prompt 让模型“只输出 JSON”很不可靠。一定要用 constrained decoding，否则解析失败率会高到没法用。
3. 标称 32K 上下文的本地模型，实际有效窗口可能远低于此。超过 8K token 的长文摘要或复杂问答，建议直接走远程。
4. 4-bit 量化对通用对话影响不大，但对指令遵循和工具调用影响明显。工具调用场景慎用低比特量化。
5. 本地部署不等于隐私安全。要关掉模型推理链路的遥测、日志，并对原始输入做脱敏，否则本地模型只是把风险从外部转移到本机日志。

## 可复用建议

- 先用静态规则跑 1-2 周，积累任务分布，再考虑加入分类模型做自动路由。
- 从抽取、摘要、分类这类任务开始本地化，通常能省下 40%-60% 的远程 token，而不是一上来追求 90%。
- 远程侧可以用便宜模型先做意图分类，再决定是本地执行、升级到更强模型，还是继续用便宜模型完成任务。
- 所有模型输出都过 schema 校验，校验失败根据任务类型决定重试、回退或中断，避免带病数据进入下游。
- 给本地模型固定一组 prompt，减少频繁变动带来的行为漂移。

## 总结

OpenClaw 多模型路由的核心是“稳定区间”：让 GPT 做规划、工具协调、长上下文推理；让本地模型做高频、低难度、隐私敏感的任务。先把边界划清楚，再逐步优化路由规则，比一次性追求全自动分配更可靠。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/7b04187980401b48.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/edbcdcbb6f1d52f8.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/c9438d5fa5952748.png)

