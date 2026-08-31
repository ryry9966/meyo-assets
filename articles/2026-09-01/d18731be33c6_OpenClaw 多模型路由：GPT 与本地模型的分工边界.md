---
title: OpenClaw 多模型路由：GPT 与本地模型的分工边界
feedId: 35629
source: 综合讨论
publishedAt: 2026-09-01
---

## 背景

OpenClaw 里把模型当成可替换的“执行器”后，很快会遇到一个工程问题：所有请求都走 GPT 类闭源 API，成本、延迟和隐私都可控性差；全部压到本地 7B/14B/32B，长上下文、多步工具调用和复杂规划又容易崩。于是需要一层路由。

这层路由不是模型投票，而是用很简单的规则把请求分到“远端强模型”和“本地可高频调用的模型”。在 OpenClaw 的多模型配置、MCP 工具和插件链路里，这个分流点可以放在 agent 执行前，也可以放在子任务级别。

## 问题

实际跑起来，最容易出问题的不是模型本身，而是“用错地方”：

- 把需要连续 5 步以上工具调用、需要根据中间结果改计划的场景交给本地模型，结果 JSON 参数错误、漏步骤。
- 把大量明确、重复、低敏感的任务交给 GPT，成本上升，延迟不稳定。
- 路由规则太细，后续维护比 prompt 还重。
- 同一套 prompt 直接切换模型，输出格式漂移，下游插件解析失败。

所以需要一个可复用的分流标准，而不是每次都临时改配置。

## 做法/步骤

1. 先定三个维度：复杂度、上下文长度、数据敏感级。  
   例如：
   - 低复杂度：单步分类、摘要、格式整理、实体抽取、PII 脱敏。
   - 中高复杂度：多步工具编排、长文档推理、需要纠错重试的计划任务。
   - 敏感数据：本地优先；非敏感数据：可远端。

2. 在 OpenClaw 中配置两个 model provider：  
   远端 GPT 类 API 作为 strong/primary；本地模型通过 Ollama/vLLM 暴露成 OpenAI-compatible endpoint。不要只配一个 fallback，而是显式注册两个模型名，例如 `gpt-4o-mini` 和 `local-qwen-14b`。

3. 写一个轻量 router，别上复杂模型判断。先用规则：

   ```python
   def route(task: Task):
       if task.token_estimate > 16_000 or task.requires_planning:
           return "gpt"
       if task.tools > 3 or task.retries > 2:
           return "gpt"
       if task.data_level == "sensitive":
           return "local"
       if task.latency_budget_ms < 800:
           return "local"
       return "local"  # 默认本地，遇到失败再升级
   ```

   这套规则不超过 6 条，开始时够用。

4. 给本地模型上 constrained decoding 或 JSON grammar，至少保证工具调用参数可解析。  
   OpenClaw 侧对工具调用结果做 schema 校验，失败一次就标记该任务不适合本地，升级到 GPT 重跑。

5. 观察一周，把导致“本地失败后升级”的任务类型记录下来。那些不是规则能救的，直接前置路由到 GPT；反之，把稳定跑通的任务固化到本地。

## 踩坑点

- 本地模型不是“小 GPT”。同样 prompt 下，它可能输出正确内容但字段缺失、话多、不按 JSON 结束。务必做格式约束和解析重试。
- 别把超过本地上下文窗口的全文丢进去。长文档先切块或先检索，路由判断要发生在 token 预估之后。
- MCP 工具调用容易放大差距。需要调用 3 个以上 MCP 工具并组合结果的，优先 GPT；只调 1 个确定工具的，本地可以。
- 路由规则里不要用“关键词匹配业务意图”作为唯一判断，很容易误判。复杂度、工具数、数据等级、上下文长度更稳定。
- 远端失败和本地失败要分开处理。不要无脑互切，否则会掩盖 prompt 或工具本身的问题。

## 可复用建议

- 从“默认本地 + 命中远端规则”开始，而不是“默认 GPT + 命中本地规则”。成本曲线会好看很多，而且能逼你找出本地真实边界。
- 保留影子路由：线上先只记录路由决策，不实际切换，离线对比结果。确认收益后再放开。
- 维护一个 20-30 条的固定 eval set，每换本地模型版本都跑一遍通过率。低于阈值就调回 GPT。
- 隐私数据做显式标记，路由层对 `data_level=sensitive` 一律不走远端，除非用户明确授权。

## 总结

OpenClaw 的多模型路由本质上是成本、复杂度、隐私和延迟之间的分流。GPT 类模型适合长上下文、多步规划、高工具调用和容错要求高的任务；本地模型适合高频、低复杂度、低延迟预算和敏感数据任务。不要试图用 prompt 把本地模型逼成 GPT，也不要把所有简单任务都推给远端。先用极简规则上线，再用失败数据和 eval set 调整边界，这套路由才能长期可维护。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/4317ddf349eb669c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/e7a1cc023277b125.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/4f05578b3dbeb28d.png)

