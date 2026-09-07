---
title: 把大模型搬回家：家用电脑部署本地 LLM 并接入 Agent 工作流的实录
feedId: 36423
source: 综合讨论
publishedAt: 2026-09-07
---

## 背景

对 OpenClaw / Agent / MCP 用户来说，本地模型的吸引力不在"替代云端"，而在三件具体的事：隐私敏感数据不出本机、高频小任务零成本、断网或限流时兜底。这篇帖记录我在一台 16GB 显存的家用机上部署本地 LLM 并接入自动化工作流的完整过程，全部可复现。

## 问题

家用机跑模型，约束很实际：显存有限、上下文吃内存、工具调用（function calling）质量参差、长任务速度熬人。但最大的问题是期望管理——本地 8B 模型干不了旗舰级推理，很多人拿它干重活，翻车后得出"本地模型没用"的结论。先想清楚让它干什么，再谈部署。

## 做法

**1. 盘硬件。** 显存决定一切：8GB 跑 7B/8B 的 Q4 量化；16GB 上 14B；24GB 才考虑 32B。Mac 统一内存是另一条路，32GB 内存跑 14B 很舒服。先确认上限再选模型，不要倒着来。

**2. 选运行时。** 我用 Ollama，理由就一个：`localhost:11434/v1` 提供 OpenAI 兼容接口，Agent 侧改个 base_url 就能接。追求性能用 llama.cpp 的 `llama-server`，要图形界面用 LM Studio，接口都兼容，迁移成本很低。

**3. 选模型。** 中文 + 工具调用场景，Qwen 系列是稳妥默认。量化一般选 Q4_K_M，体积和质量的平衡点。重点：不是所有模型都能稳定输出工具调用 JSON，选型时优先确认模型卡标注了 tool use 支持。

**4. 接入 Agent。** base_url 指向本地端点后，手动把 context 调大（默认值太小）。先用一个最简单的 MCP 工具跑通"模型 → 工具调用 → 回填 → 再生成"的完整循环，再谈复杂场景。

**5. 分层路由。** 最值得复制的架构：本地小模型做高频廉价任务（分类、抽取、摘要、格式转换），云端大模型留给规划和复杂推理。本地不是替代，是分流。

## 踩坑点

- **上下文静默截断。** llama.cpp 系默认 context 很小，Agent 的 system prompt 加一串 MCP 工具 schema 轻松超限，症状是模型"忘事"或瞎编，且不报错。必须显式设置 `num_ctx` / `-c`，并核对实际生效值。
- **KV cache 吃显存。** context 开到 16K 后，显存大头不是权重而是 KV cache。开 flash attention、KV cache 用 q8_0 量化，能省一大块。
- **工具调用格式漂移。** 部分模型偶发不按 schema 出 JSON，Agent 侧务必做校验 + 一次重试，不要信任原始输出。
- **小模型复读。** temperature 别设 0，开 repeat penalty；超过三步循环的 Agent 任务，谨慎交给本地模型。
- **Embedding 别偷懒。** RAG 的向量化用独立 embedding 模型（如 bge-m3），别指望生成模型兼任。

## 可复用建议

- 把运行时包成服务（systemd / Docker），加健康检查，Agent 启动前先探测端点。
- 给每个测过的模型记一张"能力卡"：实测 context 上限、工具调用成功率、tok/s、翻车案例。比通用跑分榜有用得多。
- 用自己的 Agent 循环做基准，测端到端任务完成率，而不是只看社区榜单。
- 配置即代码：一个 compose 文件或脚本钉死模型版本和参数，别人才复现得动。

## 总结

家用电脑跑本地 LLM 如今已是成熟选项，前提是对硬件上限有清醒认知，并把模型能力当作工程参数而非信仰。我的结论：8B 级模型 + 严格校验 + 分层路由，足以覆盖自动化工作流中大部分辅助任务；把省下的 token 预算和时间，留给真正需要强推理的环节。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/ddf12b63806fb33c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/a152e18a6361990a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/4cc92ccac2a93325.png)

