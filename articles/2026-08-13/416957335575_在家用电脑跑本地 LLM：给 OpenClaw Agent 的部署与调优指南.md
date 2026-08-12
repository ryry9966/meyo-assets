---
title: 在家用电脑跑本地 LLM：给 OpenClaw Agent 的部署与调优指南
feedId: 32870
source: 综合讨论
publishedAt: 2026-08-13
---

## 背景

OpenClaw/Agent/MCP 插件链路的常见做法是把 LLM 接到云端 API，但会碰到成本、隐私、网络稳定性问题。随着 7B~14B 模型在消费级硬件上可用，把本地 LLM 作为 Agent 底座或降级方案，逐渐有工程价值。本文记录我在家用电脑上部署本地模型并接入 OpenClaw 类工具链的实践，重点不是“能跑”，而是“能稳定跑工具调用”。

## 问题

本地部署不是装完 Ollama 就结束。实际接入 Agent 后容易遇到：工具调用 JSON 不稳定、上下文截断、显存/内存不足、并发请求排队、OpenAI 兼容层字段不一致。这些问题在纯聊天场景不明显，但一接 MCP 或多工具调度就会被放大。

## 做法与步骤

### 1. 硬件与模型选择

先按硬件筛模型，避免盲目上大模型：

- 16GB 内存、无独显：3B Q4 级别，如 Qwen2.5-3B-Instruct。
- 8GB 显存：7B Q4/Q5，如 Qwen2.5-7B-Instruct、Llama-3.1-8B-Instruct。
- 12~16GB 显存：14B Q4/Q5，或 7B FP16。
- 24GB 显存：14B/32B Q4，可跑更稳的工具调用。

优先选指令模型，并确认其 function calling 能力。不要只看通用对话榜单，工具调用稳定性才是关键。

### 2. 推理后端

起步用 Ollama，它提供 `/v1/chat/completions`，和 OpenAI SDK 基本兼容。LM Studio 适合 GUI 验证模型效果，llama.cpp/vLLM 适合后续压并发或做 grammar 约束。

Ollama 里用 Modelfile 固定参数：

```modelfile
FROM qwen2.5:7b-instruct-q4_K_M
PARAMETER temperature 0.2
PARAMETER num_ctx 8192
PARAMETER stop "</tool_calls>"
```

### 3. 接入 OpenClaw Agent / MCP

在 Agent 配置里把 LLM 的 `base_url` 指向：

```text
http://localhost:11434/v1
```

`api_key` 填任意非空字符串即可。模型名必须与 `ollama list` 里显示的名称完全一致。MCP 工具链如果通过 LLM 调用，同样复用该 endpoint。注意本地模型对某些 OpenAI 高级参数支持不完整，遇到 `parallel_tool_calls` 或 `logprobs` 报错时，先在客户端关闭相关选项。

### 4. 验证工具调用

先用单个简单工具测试，确认模型能返回可解析的 JSON，再逐步增加工具数量和嵌套参数。不要一上来就挂完整的 MCP server。

## 踩坑点

- **显存不足导致 CPU offload**：Ollama 加载时会自动把部分层放到 CPU，速度急剧下降。观察加载日志，若 offload 层数过多，应减小模型或量化等级。
- **上下文截断**：默认 `num_ctx` 较小，Agent 多轮工具调用很容易打满。设 8192 或更高，但显存占用也会上升。
- **工具调用 JSON 不稳定**：小模型或低量化模型容易输出多余文本、不闭合 JSON，或调用不存在的工具。建议降低 temperature 到 0~0.3，给 few-shot 示例，并把 tool schema 写简洁。对更严格场景，可上 llama.cpp 的 grammar 约束。
- **并发与超时**：Ollama 默认并发很有限，多个 Agent/MCP 同时请求会排队。若必须高并发，考虑 vLLM，或在客户端限制并发数。
- **API 兼容差异**：Ollama 会忽略或报错部分 OpenAI 字段。先查后端日志，不要直接在客户端盲目重试。
- **停止词缺失**：部分模型在生成 tool call 后会继续“编”内容。设置合适的 `stop` 序列可减少这种情况。

## 可复用建议

1. **先小模型打通链路**：用 1.5B/3B 确认 OpenClaw 能连通、工具格式正确，再换大模型。
2. **固定模型版本与量化**：写进启动脚本或 Docker Compose，避免模型升级改变行为。
3. **为本地模型写专用 system prompt**：加入“你只能调用提供的工具，输出 JSON，不要解释”等约束。
4. **精简工具描述**：本地模型响应较慢，MCP 工具描述过长会增加错误触发和延迟。
5. **监控资源**：常用 `nvidia-smi`、`ollama ps` 或 `/api/ps` 查看显存和 KV cache 占用。
6. **控制常驻与冷启动**：设置 `OLLAMA_KEEP_ALIVE` 保持模型常驻，减少重复加载。

## 总结

本地 LLM 可以作为 OpenClaw Agent 的实用底座，但不能把云端 prompt 直接搬过来用。真正的工作量在工具调用适配、上下文管理、并发控制和模型版本固定。按硬件选型、从单工具验证开始、做好监控与降级，才能稳定跑在本地。

---

