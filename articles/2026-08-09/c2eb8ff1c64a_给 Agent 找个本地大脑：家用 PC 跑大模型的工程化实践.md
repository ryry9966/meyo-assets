---
title: 给 Agent 找个本地大脑：家用 PC 跑大模型的工程化实践
feedId: 32216
source: 综合讨论
publishedAt: 2026-08-09
---

## 背景

在构建 Agent 自动化流水线时，很多场景并不适合直接把请求丢给云端 API：内网数据不能出域、高频调用成本急速膨胀、工具调用（Function Calling）延迟不稳定，或者你需要完全控制模型行为来适配 MCP 服务器。把大模型部署到自己的台式机或笔记本上，让它成为 OpenClaw 这类智能体框架的“本地大脑”，正在从“玩具级实验”变成可落地的方案。

这篇文章面向已经熟悉 OpenClaw、Agent 插件开发或 MCP 协议的用户，分享一套可复现的本地 LLM 部署流，重点不是跑起来，而是作为 Agent 推理后端稳定工作。

## 你会遇到的问题

家用硬件跑大模型，卡点不在模型下载，而在工程适配：

- **显存与内存瓶颈**：14B 以上模型在家用显卡上几乎无法全精度运行，必须靠量化。量化后模型对复杂工具调用格式的遵循能力会下降。
- **推理速度影响 Agent 决策链**：Agent 一次任务可能调用 LLM 十几次，若单次延迟超过 5 秒，整体任务就会超时或用户体感崩坏。
- **Ollama 等运行时的隐藏限制**：默认并发、上下文窗口、提示词缓存策略都可能和 Agent 框架的工作方式冲突。
- **模型不支持原生工具调用**：很多优秀的本地模型（如 Qwen、DeepSeek、Mistral 系列）虽然性能强，但需要你自定义提示工程或使用约束解码才能稳定输出 JSON 工具调用，否则 Agent 会卡死。

## 做法 / 步骤

### 1. 硬件基线选择
建议起点：一张 12GB 以上显存的显卡（RTX 3060 12G、4060 Ti 16G 或二手 3090），或者 32GB 以上系统内存跑 CPU 推理。纯 CPU 能跑 7B-13B 的 Q4 量化模型，但单 token 延迟在 2~10 秒，只适合非实时离线任务。我的实验环境是 i7-13700 + 32GB DDR4 + RTX 4070 12G，主要测试 7B~14B 模型。

### 2. 选型：Ollama 做推理后端
Ollama 封装了 llama.cpp，支持 GGUF 量化模型，提供与 OpenAI 兼容的 API（/v1/chat/completions），这对 OpenClaw 直接集成至关重要。
安装后先用最小模型验证链路：
```bash
ollama run qwen2.5:7b-instruct-q4_K_M
```
测试工具调用能力：通过 API 发送带 function 描述的请求，观察输出格式是否正确。如果不理想，换一个模型或调整 system prompt。

### 3. 制作适配工具的 Modelfile
对 Agent 场景，必须固化工具调用规范。新建 Modelfile（例如 `agent-modelfile`）：
```dockerfile
FROM qwen2.5:7b-instruct-q4_K_M
SYSTEM """
你是一个助手，严格按 JSON 格式调用工具。可用工具定义如下：
{tool_schemas}
如果不需要调用工具，回复普通文本。
输出格式：{"name": "tool_name", "arguments": {...}}
"""
PARAMETER num_ctx 8192
PARAMETER temperature 0.1
PARAMETER top_p 0.9
```
然后创建模型：
```bash
ollama create agent-local -f ./agent-modelfile
```
这样做的好处是，你可以为不同 Agent 角色预制不同的 Modelfile。

### 4. 与 OpenClaw 和 MCP 集成
OpenClaw 的 LLM 后端配置成 OpenAI 兼容模式，base_url 指向 `http://localhost:11434/v1`，api_key 随意填（Ollama 本地不需要鉴权），模型名填 `agent-local`。MCP 服务器作为工具提供方，OpenClaw 会自动从 MCP 服务器拉取工具定义并拼进系统提示。难点在于，你得确认 Ollama 侧的系统提示里预留了正确的 `{tool_schemas}` 占位符，让 OpenClaw 能够注入实时工具列表。如果模型输出格式偶尔偏差，开启 OpenClaw 的输出解析重试机制即可。

### 5. 性能微调
- 设置 `OLLAMA_NUM_PARALLEL` 环境变量为 1，避免多请求争抢显存导致推理停顿。
- 通过 NVIDIA 控制面板或 `nvidia-smi` 锁频，固定功耗墙，防止温度高导致降频。
- 限制上下文：Agent 历史对话很快撑爆 8192 token 窗口，你需要在 OpenClaw 中启用上下文精简策略。

## 踩坑点与排障

1. **“模型突然输出乱码或循环”**：大概率是因为量化版本与原始模型不匹配，或者使用了过低的 K/V 量化精度。优先选择带 `q4_K_M` 或 `q5_K_M` 标签的 GGUF，不要用纯 `q4_0`。
2. **Ollama 服务自动卸载模型**：默认空闲 5 分钟后模型被卸载。在启动时加 `OLLAMA_KEEP_ALIVE=-1` 环境变量让模型常驻。
3. **工具调用返回格式被截断**：推理时 `num_predict` 默认限制可能不够。修改 Modelfile 增加 `PARAMETER num_predict 2048`。
4. **Agent 调用链超时**：单次生成耗时需控制在 3 秒内，否则重试会导致雪崩。实测 Qwen2.5:7b-q4_K_M 在 RTX 4070 上平均首 token 延迟 0.4s，生成 200 token 的工具调用约 2.1s，可以接受。若使用 14B 模型，最好有 16GB 以上显存，或回退到 CPU 混合推理但要接受 3~5 倍延迟。
5. **MCP 工具列表过长撑爆系统提示**：OpenClaw 可选择仅发送与当前意图相关的工具子集，配置 `tool_filter` 或包装 MCP server 输出。

## 可复用建议

- 从 7B 模型开始，验证完 Agent 的完整控制环再往上换。7B 工具调用能力在多数 Qwen2.5、Mistral Nemo 模型上已经够用。
- 构建一个轻量脚本监控 Ollama 的 `/api/tags` 和 `ps`，方便一键检查模型状态。
- 在 OpenClaw 的配置里，把 `llm.request_timeout` 设置为 30s 或更高，避免本地推理高负载时被误判为失败。
- 如果你需要在同一台机器上同时跑 Agent、MCP 服务器、本地 LLM，使用 `taskset` 或 CPU 亲和性把服务进程隔离，防止上下文切换拖慢推理线程。

## 总结

在家用 PC 上给 OpenClaw Agent 挂一个纯本地的大模型，经过合理的量化和工程适配后，完全可以承担自动化任务的核心推理工作。它带来的隐私可控、零 API 成本以及离线可用性，会让你的 MCP 工具链更加健壮。先跑通一个 7B 工具调用链路，再根据任务复杂度逐步升级模型，是成本最低的实践路径。

---

