---
title: 本地 LLM 部署：在家用电脑上跑大模型的实践指南
feedId: 35369
source: 综合讨论
publishedAt: 2026-08-30
---

## 背景与问题

对 OpenClaw、Agent、MCP 和自动化脚本用户来说，本地 LLM 最大的价值不是“免费聊天”，而是把模型服务拉回自己的网络边界内：调试 MCP 工具链路时不被限流，跑批处理时不受外部 API 波动影响，处理敏感配置时不离开机器。

但家用电脑跑大模型有个容易忽略的问题：能加载模型不等于能接住 Agent 的工作负载。Agent 需要函数调用、长上下文的工具结果、稳定的结构化输出，这些比单纯对话更吃显存和配置。很多教程只到“启动一个 WebUI 聊天”就结束了，离自动化使用还差一截。

## 实践步骤

### 1. 先按显存选模型档位

不要一上来拉 70B。家用常见的 8GB 显存建议 7B Q4_K_M 或 7B Q5；16GB 可上 14B Q4_K_M；24GB 可尝试 32B Q4 或 14B Q8。中文场景优先选 Qwen 系列，并确认是 Instruct 且注明支持 function calling 的版本。模型大小要留出 KV cache 和工具输出空间，不能把显存卡满。

### 2. 用 Ollama 起 OpenAI-compatible 服务

Ollama 对家用环境最省事。拉取模型后不要用默认聊天 UI 作为最终形态，而是把它当后端服务：

```bash
OLLAMA_NUM_PARALLEL=1 OLLAMA_MAX_LOADED_MODELS=1 ollama serve
```

然后在 OpenClaw 的模型配置里把 `base_url` 指向 `http://127.0.0.1:11434/v1`，`api_key` 可填任意值。关键是把 `num_ctx` 设到 8192 或更高，默认窗口太小，工具返回一多就截断。

### 3. 用最小工具调用测试验收

先别直接接复杂 MCP。用一个简单函数调用验证模型是否按要求返回结构化数据：

```json
{"name":"get_weather","arguments":{"city":"shenzhen"}}
```

如果模型输出前后夹带解释、多个 JSON 块或漏字段，要么换支持函数调用的模型，要么在 Agent 侧做宽松解析；对生产自动化，最好用支持 grammar/constrained decoding 的服务端。

### 4. 控制上下文和工具结果

本地模型最怕上下文膨胀。MCP 工具返回的网页、日志、搜索结果要截断，工具描述尽量短，不要把整篇文档塞进 system prompt。否则 8GB 显存跑 7B 模型，KV cache 会先 OOM，尤其是长对话。

## 踩坑点

- **启动时正常，长任务崩溃**：多半是上下文超限。默认 `num_ctx` 太低或工具结果没有截断。
- **工具调用 JSON 不稳定**：小模型温度要低，`temperature` 设 0 或 0.1；不要用 base 模型。
- **CPU 推理勉强能跑但自动化超时**：不是所有本地模型都适合实时 Agent。批处理可等，交互需 GPU。
- **显存被多个模型占满**：Ollama 默认可能加载多个模型，设置 `OLLAMA_MAX_LOADED_MODELS=1`。

## 可复用建议

把本地模型当成一个可替换的服务，而不是写死在插件里。建议用环境变量配置 `LOCAL_LLM_BASE_URL` 和 `LOCAL_LLM_MODEL`，并准备一个最小测试脚本：普通对话、单工具调用、多工具选择、JSON 输出。每次换模型或升级后跑一遍，比手动聊天可靠得多。

对于 MCP 用户，可以在工具结果进入模型前做一层“摘要/截断”，保留状态码、关键字段和第一条错误信息，而不是原始 HTML 或几千行日志。这样 7B 模型也能在不少自动化场景里稳定工作。

## 总结

本地 LLM 在家用电脑上跑通不难，难的是让它稳定服务于 Agent 和自动化。选对量化档位、限制上下文、控制工具返回、用最小测试验收，比追求更大参数更有回报。先跑小模型，再逐步扩展，能少踩很多坑。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/752317a44211c8c2.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/20379785fb2944df.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/2385dcfe4785f1e6.png)

