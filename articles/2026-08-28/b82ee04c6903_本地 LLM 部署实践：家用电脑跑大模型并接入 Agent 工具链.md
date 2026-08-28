---
title: 本地 LLM 部署实践：家用电脑跑大模型并接入 Agent 工具链
feedId: 35023
source: 综合讨论
publishedAt: 2026-08-28
---

## 背景

做 OpenClaw/Agent/MCP/插件自动化时，很多任务并不适合直接上云：日志与隐私数据脱敏、内网信息抽取、长期运行的固定流程、需要完全复现的实验环境。家用电脑部署本地 LLM，可以作为一条可控的 OpenAI 兼容推理后端，让 Agent 主控或子任务走本机完成。

但本地部署不是“换一个 API 地址”那么简单。它需要管理显存、量化格式、chat template、工具调用稳定性，以及与 OpenClaw/MCP 的接入约束。

## 问题

家用电脑常见硬件：NVIDIA 8GB/16GB 显卡、Apple Silicon 统一内存、纯 CPU 主机。主要问题有：

- 显存或统一内存不足，长上下文容易 OOM。
- 量化模型能聊天，不代表能稳定 function calling。
- Agent/MCP 工具调用对输出格式和指令遵循要求更高，本地小模型容易退化。
- 本地推理并发能力弱，OpenClaw 多个任务同时请求时容易超时或崩溃。

因此，本地 LLM 更适合作为“执行范围明确的子任务后端”，而不是一开始就完全替换云端主控。

## 做法/步骤

### 1. 确认硬件并选择运行时

有 NVIDIA GPU 优先用 Ollama，Windows/Linux 安装最省事；需要细粒度控制时可用 llama.cpp server。Apple Silicon 设备用 Ollama 或 LM Studio 即可。纯 CPU 主机只建议跑 7B Q4 以下模型，速度会明显偏慢。

### 2. 选择合适模型

建议从 7B–14B 指令模型开始：Qwen2.5-7B/14B-Instruct、Llama-3.1-8B-Instruct 等。参考配置：

- 8GB 显存：7B Q4_K_M 或 Q5_K_M，上下文控制在 8k。
- 16GB 显存：14B Q4_K_M 或 Q5_K_M，上下文 8k–16k。
- 32GB 统一内存的 Mac：可尝试 32B Q4_K_M，但并发仍要限制。

Agent 场景下，工具调用模型尽量使用 Q5_K_M 及以上量化，4-bit 模型退化更明显。

### 3. 启动 OpenAI 兼容 API

Ollama 默认暴露：

```text
http://127.0.0.1:11434/v1
```

llama.cpp server 需要显式指定 chat template：

```bash
llama-server -m model.gguf \
  --jinja --chat-template chatml \
  --host 127.0.0.1 --port 8080
```

先通过 `/v1/chat/completions` 做最小请求测试，确认返回格式正常。

### 4. 接入 OpenClaw/MCP

在 OpenClaw 中把模型提供方指向本地 OpenAI 兼容地址，模型名使用实际 tag，例如 `qwen2.5:14b-instruct-q5_k_m`。为本地模型单独建 agent 或 route，并设置：

- 并发：1，最多 2
- 超时：90–120 秒
- 温度：0.1–0.3
- MCP 工具先挂 2–5 个，不要一次接入过多工具

### 5. 做能力验证

准备固定测试集：要求模型输出 JSON、调用一个简单 MCP 工具、忽略无关工具。只有通过真实工具调用回归测试，才适合放进自动化流程。

## 踩坑点

- **显存不够**：长上下文的 KV cache 占用非常可观。不要直接给 32k，先从 8k/16k 起步。7B Q5_K_M 在 8GB 显卡上配 8k 上下文可能已经很紧。
- **工具调用退化**：量化后 function calling 准确率可能显著下降，尤其 Q4 或以下。复杂 Agent 任务建议 Q5_K_M 以上，或者让本地模型只做意图分类、摘要、脱敏，复杂工具调用仍走云端。
- **模板错误**：llama.cpp 加载 GGUF 时如果不指定 chat template，输出会变成续写而非对话，Qwen 系列尤其常见。模型卡必须记录模板来源。
- **并发崩溃**：家用显卡同时处理多个长请求容易 OOM 或延迟暴涨。OpenClaw 中限制并发为 1，并监控显存。
- **把聊天能力误当工具能力**：很多模型通用 benchmark 不错，但 function calling 不稳定。用真实 MCP 工具做回归测试，而不是只测闲聊。

## 可复用建议

- 本地服务用 systemd 或 Docker Compose 管理，固定端口、模型 tag、上下文长度，保证可复现。
- 记录模型卡：模型名、量化、context、chat template、温度、已验证的 MCP 工具列表。
- 给本地模型写 system prompt 时减少模糊指令，工具参数尽量用字符串或数字，避免复杂嵌套对象。
- 使用 Ollama 的 `format: json` 或 llama.cpp 的 grammar/json schema 约束，提高结构化输出稳定度。
- 准备健康检查脚本：调用 `/v1/models` 和一次最小工具调用，失败时告警或自动重启。
- 自动化任务分级：隐私清洗、内网信息抽取等适合本地；复杂多步推理保留云端主控，本地模型做协作者。

## 总结

本地 LLM 部署能否用起来，关键不在“能跑”，而在“可接入、可复现、可控制”。对 OpenClaw/MCP 用户来说，更务实的路径是：选对量化模型、固定 chat template、限制并发与上下文、用真实工具调用做回归。先把本地模型当作一个执行稳定的小工具，比一步到位替换云端主控更现实。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/87ee84d5d36da06e.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/bff09afb16c63c7f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/2d02654ca9e63bba.png)

