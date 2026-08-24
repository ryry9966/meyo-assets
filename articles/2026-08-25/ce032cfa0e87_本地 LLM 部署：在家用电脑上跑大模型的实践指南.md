---
title: 本地 LLM 部署：在家用电脑上跑大模型的实践指南
feedId: 34573
source: 综合讨论
publishedAt: 2026-08-25
---

# 本地 LLM 部署：在家用电脑上跑大模型的实践指南

## 背景

在 OpenClaw/Agent 的自动化链路里，很多时候需要一个“内部模型节点”：处理带内部信息的任务、离线跑批、调试 MCP 工具调用，或者不想把每次请求都发到云端 API。家用电脑部署本地 LLM 是实现这个目标的一种低成本方式。它不能替代高并发生产环境，但适合个人实验、旁路自动化和原型验证。

## 问题

真正的问题不是“能不能跑”，而是“跑起来之后能不能稳定接进 Agent/MCP 链路”。常见问题包括：

- 显存/内存不够，加载失败或速度极慢。
- 小模型对函数调用和 JSON 输出不稳定。
- 推理框架与 OpenClaw 的接口不一致。
- 多请求并发时排队严重。
- 上下文设置不合理导致显存溢出。

## 做法/步骤

### 1. 先评估硬件与任务

显存决定了模型规模上限。粗略估算：7B 模型 Q4 量化约占用 4-5GB，14B Q4 约 9-10GB，32B Q4 约 20GB。如果只是做轻量 agent、文本分类、工具参数提取，7B-14B 足够了。

### 2. 选择模型与格式

建议从 Qwen2.5-7B-Instruct 或 Qwen2.5-14B-Instruct 开始，中文和工具调用表现较稳定。格式优先选 GGUF，方便在不同设备上切换。

### 3. 选择推理引擎

- **快速起步**：Ollama。安装后 `ollama pull qwen2.5:7b-instruct-q4_K_M`，默认监听 `11434`，自带 OpenAI 兼容接口。
- **可控部署**：llama.cpp server。适合需要调整 `--n-gpu-layers`、`-c` 上下文、`--threads` 的场景。
- **高吞吐**：vLLM 或 SGLang。适合有 CUDA 环境、需要真正并发和指标监控的情况。

### 4. 接入 OpenClaw/MCP

Ollama 或 vLLM 提供 OpenAI 兼容端点后，在 OpenClaw 里只需把 `base_url` 指向本地服务，`api_key` 填任意非空值即可。MCP 工具调用时，建议把 `temperature` 设为 `0.1-0.3`，并在工具描述里写清参数类型和约束。

### 5. 设置上下文与资源限制

不要盲目开长上下文。根据任务需要设置 `num_ctx`，例如 4096 或 8192。长上下文会让 KV cache 迅速增长，家用显卡容易爆显存。并发场景建议在 OpenClaw 侧加队列或信号量，避免同时发多个请求。

## 踩坑点

- **工具调用不稳定**：小模型可能不按 JSON schema 输出。可以开启 JSON mode / response_format，减少自由文本，并在系统提示里给出一个工具调用示例。
- **量化损失**：Q4_K_M 通常够用，但 Q2/Q3 可能导致中文逻辑变差。关键路径用 Q5_K_M 或 Q6。
- **纯 CPU 运行**：可以跑，但 token 速度可能只有 2-10 tok/s，适合离线批处理。llama.cpp 多线程能改善。
- **Windows/WSL2**：Ollama 在 Windows 上可能默认用 CPU；想用 NVIDIA GPU 最好用 WSL2 或原生 Linux 部署。
- **显存不足/溢出**：加载时加上 `--n-gpu-layers` 控制 GPU offload 层数；或者减小上下文、降低并发。
- **OpenClaw 侧超时**：本地推理首 token 延迟较高，需要把请求超时调大，否则 agent 可能误判失败。

## 可复用建议

1. 把本地 LLM 当作“能力受限的内部服务”，保留云端 fallback 入口。
2. 统一通过 OpenAI 兼容接口接入，OpenClaw 配置一个 `local` 模型环境，方便切换。
3. 写一个健康检查脚本，定期请求 `v1/models` 并记录延迟和显存。
4. 为每个模型记录量化版本和部署参数，例如 `qwen2.5:7b-instruct-q4_K_M` + `num_ctx=8192`。
5. 对提示词做本地化压缩，减少无用 token，提升速度。
6. 先用最小闭环跑通：Ollama + 7B + 一个工具调用用例，再逐步扩大模型和复杂度。

## 总结

在家用电脑上部署本地 LLM，对 OpenClaw/Agent/MCP/自动化实践者来说是可行的。关键是合理管理硬件预期、选择合适量化和推理引擎、保持接口标准化，并把工具调用和并发作为重点调优对象。先跑通最小闭环，再根据实际负载决定是否升级模型或迁移到更高吞吐的部署方案。这样得到的不是一个演示品，而是一个能稳定接进自动化链路的本地模型节点。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/9b7306dc47a1dde1.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/b7fb1ed014bf075d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/f6b282199df7bcda.png)

