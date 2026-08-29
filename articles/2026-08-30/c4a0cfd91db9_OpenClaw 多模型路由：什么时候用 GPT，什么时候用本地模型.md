---
title: OpenClaw 多模型路由：什么时候用 GPT，什么时候用本地模型
feedId: 35292
source: 综合讨论
publishedAt: 2026-08-30
---

## 背景

OpenClaw 接入 MCP、插件和自动化任务后，模型调用会从“只接一个模型”变成“每个任务都要选路”。常见做法是两类：要么所有任务都上 GPT，成本高、延迟大，遇到内部文档还可能泄隐私；要么全压本地 7B/14B，简单抽取还行，一碰到多步规划、复杂工具编排就输出不稳。

问题不是“哪个模型更好”，而是怎么让任务走对路。

## 问题

多模型路由的关键变量不是模型热度，而是任务形状：

- 这个任务边界是否清晰？
- 上下文有多长？
- 是否包含敏感数据或需要离线？
- 输出是否要求严格 JSON / 工具调用？
- 是否需要多步推理和规划？

把这些变量做成规则，比人工每次判断可靠得多。

## 做法/步骤

### 1. 先给任务分级

适合本地模型：

- 文本抽取为 JSON
- 短文本分类
- 格式转换
- 敏感信息处理
- 高频本地自动化
- 简单工具选择

适合 GPT 类云模型：

- 多步任务规划
- 模糊意图理解
- 长文档综合
- 代码调试
- 复杂 MCP 工具编排
- 对未知工具输出做推理

### 2. 在 OpenClaw 里加路由配置

不要硬编码关键词，建议用规则组合。例如：

```yaml
route:
  - name: pii_or_offline
    when: input.has_pii or not network.online
    use: local

  - name: structured_short
    when: task.kind in [extract, classify, normalize] and input.tokens < 2000
    use: local

  - name: complex_planning
    when: task.kind in [plan, debug, code_gen, multi_tool] or input.tokens > 8000
    use: gpt

  - name: invalid_local_escalation
    when: local.output.schema_valid == false and local.retries >= 2
    use: gpt
```

实际实现时，可以把路由做成 OpenClaw 插件，输入包括任务类型、输入 token、MCP 工具数量、是否离线、是否含 PII，以及上一次本地模型输出是否解析成功。

### 3. 本地模型要当成工程组件约束

- 用 temperature 0
- 开启 JSON schema / grammar 约束
- 系统提示尽量短
- 只放本次任务相关的工具 schema，不要塞全量 MCP 工具
- 长上下文先做截断或先摘要

### 4. 记录路由结果

至少记录：任务 ID、命中的路由规则、使用模型、输入/输出 token、耗时、是否解析成功、是否升级到 GPT。没有这些统计，无法判断哪条规则在浪费云额度或制造失败。

## 踩坑点

1. **本地模型工具调用时好时坏**  
   不要看到“支持 function calling”就直接执行。先校验 tool_call JSON，解析失败先重试，再失败升级。

2. **上下文盲区**  
   本地模型实际可用 context 可能只有 4K/8K，MCP 工具 schema 会占掉一大半。要按 token 路由，不要按字符数估算。

3. **不要用模型大小做绝对判断**  
   7B 做短文本抽取可能很稳，做多步工具选择可能很差。必须用自己的 eval 集验证。

4. **过度升级**  
   本地模型第一次输出不对就转 GPT，成本很快又回到全量 GPT。设固定重试和严格 prompt，再升级。

5. **工具全量注入**  
   把所有 MCP 工具都塞给本地模型，会让输出退化。路由前先做工具预筛，只保留相关工具。

## 可复用建议

- 建 50-100 条真实任务 eval，覆盖抽取、分类、规划、工具调用、长文档。本地模型在结构化任务上稳定率高于 95% 再放量。
- 本地和云模型维护两套 prompt 模板。本地 prompt 短、直接、给 one-shot 或 few-shot；云 prompt 可以带更多上下文和推理要求。
- 路由规则做成配置或插件，不要写死在业务逻辑里，方便回滚和 A/B。
- 隐私规则优先：内部文档、密钥、PII、离线场景，直接走本地，不做混合判断。

## 总结

OpenClaw 的多模型路由本质是任务分流：边界清晰、短上下文、高隐私、高并发的任务走本地模型；需要规划、推理、长上下文和复杂工具编排的任务走 GPT 类云模型；中间加一条校验兜底，避免小模型静默失败。路由做得合理，成本、延迟和隐私都能降下来；做得草率，只会得到一个更难排障的双模型系统。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/06778ec1ef53415e.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/ac77904e36277e47.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/75f39ff354e5627e.png)

