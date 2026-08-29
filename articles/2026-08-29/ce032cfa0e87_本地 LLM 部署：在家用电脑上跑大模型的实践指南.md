---
title: 本地 LLM 部署：在家用电脑上跑大模型的实践指南
feedId: 35182
source: 综合讨论
publishedAt: 2026-08-29
---

# 本地 LLM 部署：在家用电脑上跑大模型的实践指南

## 背景

对 OpenClaw、Agent、MCP 和插件自动化实践者来说，模型后端不应该只是一个黑盒。云端 API 省事，但网络抖动、限流、隐私和调试不可控会直接影响自动化链路。把模型跑在本地，意味着可以离线执行、处理敏感数据、自由抓包和调整参数，更适合个人 Agent、工具调用和 MCP 实验。

## 问题

家用电脑的约束很直接：显存和内存有限，大模型加载不了；量化可以降低门槛，但可能削弱指令遵循和工具调用能力；Agent 不是聊天，模型输出需要稳定触发 `tool_calls`；长时间任务上下文膨胀会导致复读、截断或超时。因此，本地部署的目标不是“跑最大的模型”，而是先跑通一条可控的工具调用链路。

## 做法/步骤

### 1. 硬件基线

- NVIDIA 显卡 8GB VRAM 起步：7B/8B 模型 Q4_K_M 约 4-6GB，可留出上下文和 KV cache。
- 无独显：32GB 内存 CPU 可以跑 7B Q4，但首 token 慢，适合调试，不宜做长任务。
- macOS Apple Silicon：使用 Metal/MPS，16GB 内存可以跑 7B/8B 模型。

### 2. 推理服务选型

建议先用 Ollama，一条命令即可提供 OpenAI 兼容 API。若要图形化调参可换 LM Studio，要脚本化或嵌入可上 llama.cpp。家用环境不必引入 vLLM/TGI。

### 3. 部署步骤

```bash
ollama pull qwen2.5:7b-instruct-q4_K_M
ollama serve
```

默认地址为 `http://127.0.0.1:11434`。验证服务：

```bash
curl http://127.0.0.1:11434/api/tags
curl http://127.0.0.1:11434/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"qwen2.5:7b-instruct-q4_K_M","messages":[{"role":"user","content":"ping"}]}'
```

在 OpenClaw 中配置 OpenAI 兼容 provider：`base_url` 填 `http://127.0.0.1:11434/v1`，`model` 填 Ollama 实际模型名，`api_key` 可随意。然后打开工具调用/MCP，先做单步测试，再跑多步任务。

### 4. 参数建议

- `temperature`：0 或 0.1，工具调用更稳定。
- `num_ctx`：4096 或 8192，先保守设置，OOM 再降。
- `num_predict`：限制输出长度，防止复读和失控。
- system prompt 明确“只返回一个工具调用，JSON 必须闭合”。

## 踩坑点

- **显存 OOM**：不要只看参数量，KV cache 和 context 会额外吃显存。用 `nvidia-smi` 和 `ollama ps` 观察。
- **工具调用格式不兼容**：模型可能输出多余解释、不闭合 JSON 或漏掉 `tool_calls`。确保 model ID 和 provider 配置一致；必要时用 grammar/约束解码，或更换工具调用微调模型。
- **上下文膨胀**：MCP 工具返回过长会挤爆 context。对工具输出做裁剪，保留关键字段；多轮任务定期摘要或重置。
- **超时**：CPU 推理首 token 很慢，OpenClaw 侧超时需要调大；也可先发一条 warmup 请求。
- **环境差异**：Windows 上 Ollama 模型目录和 WSL 路径不同，备份模型时要注意。
- **驱动问题**：NVIDIA 驱动过旧会导致 CUDA 不可用，确认启动日志中有 GPU offload。

## 可复用建议

- 用 Docker Compose 固化 Ollama + OpenClaw，模型目录挂载到卷。
- 写一个工具调用冒烟测试：发送带 tools 的请求，检查返回中是否存在合法 `tool_calls`。
- 将模型配置（量化、context、温度、provider）纳入版本管理，方便回滚。
- MCP 工具描述写清参数类型、返回结构和副作用，模型调用准确率会明显提升。
- 保留推理日志，监控显存、CPU 和 token 速度。

## 总结

本地 LLM 部署的核心不是“跑大模型”，而是用可控的模型后端支撑 Agent、MCP 和自动化链路。先小模型验证工具调用，再按显存升级；稳定参数和清晰工具描述比盲目堆参数量更有效。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/8db5e4fb3ef67541.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/be631d16a5362815.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/65a1a56a00ccd50b.png)

