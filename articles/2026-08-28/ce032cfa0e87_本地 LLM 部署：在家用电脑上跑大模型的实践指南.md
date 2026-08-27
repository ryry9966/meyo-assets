---
title: 本地 LLM 部署：在家用电脑上跑大模型的实践指南
feedId: 34978
source: 综合讨论
publishedAt: 2026-08-28
---

## 背景

在 OpenClaw、Agent、MCP 和插件开发中，经常会遇到频繁调用模型做工具选择、JSON 生成、任务拆解的场景。云端 API 在开发调试阶段有三个明显问题：调用成本累积快、网络波动影响复现、涉及私有数据时不便上传。家用电脑上部署本地 LLM，不是为了替代生产级模型，而是给开发链路增加一个可控、离线、可反复折腾的推理后端。

## 问题

家用硬件差异很大，但最常见的坑不是“跑不起来”，而是跑起来之后工具调用不稳定、API 兼容性差、显存被 context 和 KV cache 吃满。很多教程只讲下载模型，不讲如何接入 Agent 框架，导致本地模型能聊天，却不能稳定完成 OpenClaw 需要的 tool calling。

## 做法/步骤

### 1. 先评估硬件

按 NVIDIA 显存粗略匹配：

- 8GB：7B/8B 模型 Q4 量化，context 不超过 8192
- 12GB：14B 模型 Q4 量化，或 7B 模型更长 context
- 16GB：14B Q4 + 较长 context，或 32B 低量化
- 24GB：32B Q4 或 14B 高精度，适合更复杂工具调用

内存建议至少 16GB，磁盘预留 30–50GB，因为量化权重通常单个就 4–10GB。

### 2. 选模型

优先选经过 instruct/tool 微调的版本，例如 Qwen2.5/Qwen3 系列 Instruct 版本、Llama-3.1/3.3 等。不要用 base 模型做 Agent 任务，它们对工具调用格式不敏感。本地部署时，先看该模型是否在 API 调用格式上稳定输出 `tool_calls`，而不是只看聊天流畅度。

### 3. 部署推理服务

最简单的是 Ollama，一条命令可以拉起本地服务，并提供 OpenAI 兼容端点：

```bash
ollama run qwen2.5:7b-instruct-q4_K_M
```

如果希望更细粒度控制，用 llama.cpp 的 llama-server：

```bash
llama-server -m model.gguf \
  --ctx-size 8192 \
  --n-gpu-layers 99 \
  --host 0.0.0.0 \
  --port 8080
```

其中 `--n-gpu-layers` 决定多少层放到 GPU，设置太小会让速度骤降。vLLM 适合高吞吐，但对显存和 CUDA 环境要求高，家用电脑一般不作为首选。

### 4. 接入 OpenClaw/Agent

把 Agent 的模型 base URL 指向本地服务，模型名保持与本地一致。建议先写一个最小脚本验证连通性和工具调用：

```python
client = OpenAI(base_url="http://127.0.0.1:11434/v1", api_key="local")
resp = client.chat.completions.create(
    model="qwen2.5:7b-instruct-q4_K_M",
    messages=[...],
    tools=[...],
    temperature=0,
)
```

对本地小模型，`temperature` 建议设 0 或接近 0，关闭过度采样，能提高工具调用稳定性。

## 踩坑点

- **OOM 不一定是模型太大**：很多时候是 context 设得过大，KV cache 占满了显存。先把 `--ctx-size` 降到 4096 或 8192 再排查。
- **工具调用时好时坏**：小模型对 system prompt 和输出格式非常敏感。检查 GGUF 的 chat template 是否正确；如果模板缺失，需要手动指定 `--chat-template`。
- **OpenAI 兼容端点细节不一致**：有些本地服务对 `tools`、`tool_choice`、`response_format` 支持不完整，Agent 框架发送某些字段会直接报 400。需要打开服务日志，看是哪个字段不被支持。
- **CPU 推理拖慢整个 Agent**：7B 模型在普通 CPU 上可能只有 5–10 token/s，交互式任务会很难受。尽量把支持 GPU 的层卸载到显卡。
- **下载文件不完整或磁盘不足**：量化权重量大，下载后先校验文件大小，避免运行时才发现加载失败。

## 可复用建议

- 固定一条包含工具调用的基准 prompt，分别测试 7B/14B/32B 在不同量化下的成功率、延迟和显存占用，形成自己的本地模型测试表。
- 把本地推理服务写成 systemd 或启动脚本，重启后自动恢复，避免每次手动拉起。
- 先用小模型验证 Agent 流程，再换大模型验证质量，不要一上来就下最大的权重。
- 为工具调用准备 few-shot 示例，显式展示函数选择和参数格式，能明显降低失败率。
- 家用本地推理不要用于生产高并发，只适合开发、离线、隐私敏感和低成本实验。

## 总结

家用电脑部署本地 LLM 的核心不是“追大模型”，而是把硬件评估、量化选择、推理服务、API 兼容、工具调用调优这条链路跑顺。对 OpenClaw 这类 Agent 用户来说，本地模型的价值在于离线可复现、调试成本低、数据不出本机。接受它能力上限，固定配置和测试方法，才能真正用起来。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/8d0ad5c6ddeecabc.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/dce75f606f192b33.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/ddd42e7038ce57da.png)

