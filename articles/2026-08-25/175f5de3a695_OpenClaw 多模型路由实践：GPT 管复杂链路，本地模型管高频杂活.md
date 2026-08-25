---
title: OpenClaw 多模型路由实践：GPT 管复杂链路，本地模型管高频杂活
feedId: 34733
source: 综合讨论
publishedAt: 2026-08-25
---

## 背景

在 OpenClaw 里跑 Agent、接 MCP、挂插件，很快会碰到一个现实问题：所有请求都走 GPT 成本高、延迟不稳定；全用本地模型，复杂工具调用和推理又容易翻车。多模型路由不是新概念，但在 OpenClaw 场景下，目标不是做一个大而全的路由器，而是把“高频、低风险、可失败”的任务分给本地模型，“低频、高复杂度、强工具依赖”的任务留给 GPT。

## 问题

最常见的错误是按模型名字选型，或者按“隐私/成本”一刀切。实际上决定因素应该是三个：任务失败成本、工具调用复杂度、上下文敏感度。一个简单 JSON 修复用 GPT 是浪费；一个多步浏览器自动化用 7B 本地模型，基本会卡在 MCP 的 schema 上。

## 做法/步骤

**1. 建两个 profile**

在 OpenClaw 配置里保留 `cloud` 和 `local`。`cloud` 用 GPT-4o-mini 或 GPT-4o，`local` 用 Ollama 挂 `qwen2.5:7b-instruct` 或 `llama3.1:8b-instruct`。不要频繁切换模型名，先固定版本。

**2. 设置路由预检**

用本地小模型做一次意图分类，输出 `local` 或 `cloud`。分类 prompt 只做简单判断，不处理业务。若输出非法或超时，默认走 `cloud`。

**3. 路由规则**

可以按以下字段判断：

- `task.tools` 包含 `browser`、`code_interpreter`、`multi_mcp`：走 `cloud`
- `task.input` 包含密钥、内部日志、用户隐私文本：走 `local`
- `task.max_steps > 5`：走 `cloud`
- `task.type` 属于 `summarize`、`rewrite`、`parse`、`classify`：走 `local`
- `task.complexity_score >= 4`：走 `cloud`

**4. 本地模型只给简单工具**

不要给本地模型暴露复杂 MCP 工具，尤其是嵌套对象多的 schema。可以先让它处理无工具文本任务，或只给 `read_file`、`search_notes` 这类简单工具。

**5. 设置升级与兜底**

本地模型出现 JSON 解析失败、`tool_call` 格式错误、超时或连续两次空输出时，自动升级到 `cloud`，并在 metadata 记录 fallback 原因。

## 踩坑点

- **本地模型对复杂工具调用不稳定。** 7B/8B 模型在多层嵌套 schema 下经常输出缺字段或错误类型。不要只看 benchmark，要在你的 MCP 工具上实测成功率。
- **本地不是零成本。** 显卡显存、电费、模型加载时间、并发瓶颈都会影响体验。小内存机器跑 7B 可能比 GPT-4o-mini 还慢。
- **隐私要确认全链路。** 如果本地模型接了云端 embedding、审阅插件或远程 MCP，敏感数据仍可能出网。不能只把模型换成本地就认为安全。
- **上下文长度退化。** 本地模型在 8k 以上上下文时，指令遵循能力和速度都会下降。长文档任务不要硬塞给本地。
- **路由分类本身会误判。** 如果分类模型把“复杂代码重构”判成 `local`，后续失败成本可能更高。建议敏感操作强制 `cloud`。

## 可复用建议

- **任务分级：** P0 关键路径强制 `cloud`；P1 本地先跑但失败可升级；P2 本地优先，失败不影响主流程。
- **可观测：** 在 OpenClaw 日志里记录 `model_used`、`route_reason`、`fallback`、`latency_ms`。没有这些字段，路由调优就是拍脑袋。
- **缓存与去重：** 相同或高度相似的 prompt 结果做缓存，能显著减少云端调用和本地重复计算。
- **小范围灰度：** 先让本地模型处理只读、非关键、可重试的任务，连续观察一周成功率再扩大范围。
- **版本锁定：** 本地模型升级后要重新跑一遍工具调用回归，尤其是 MCP schema 变化时。

## 总结

OpenClaw 的多模型路由，本质不是“GPT 贵所以少用”，而是让失败成本决定模型边界。复杂推理、多步工具编排、高敏感操作留给 GPT；高频解析、草稿生成、低风险分类交给本地模型。关键是设置明确的兜底和可观测，否则路由会变成新的不稳定来源。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/0c48c1b87ca87ddb.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/61a1e608c9b544c7.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/3168c70347bca6fc.png)

