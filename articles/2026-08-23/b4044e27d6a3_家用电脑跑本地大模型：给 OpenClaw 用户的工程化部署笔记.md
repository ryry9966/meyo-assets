---
title: 家用电脑跑本地大模型：给 OpenClaw 用户的工程化部署笔记
feedId: 34278
source: 综合讨论
publishedAt: 2026-08-23
---

# 家用电脑跑本地大模型：给 OpenClaw 用户的工程化部署笔记

## 背景

很多 OpenClaw/Agent 用户想把推理后端从云端 API 换成本地模型。原因很实际：长时间自动化任务 token 成本高、涉及隐私数据不想出本机、断网环境需要可用、实验需要可复现。但家用电脑硬件有限，显存和内存是主要瓶颈。这篇文章记录我在普通家用机器上跑本地 LLM 并接入 OpenClaw 的过程，重点不是“跑起来”，而是“能稳定驱动 Agent”。

## 问题

本地部署不是下载一个模型那么简单。对 OpenClaw 用户来说，有三个核心问题：

1. 模型能不能在有限显存下跑起来，推理速度是否可接受；
2. 小模型是否具备 Agent 需要的工具调用和多步规划能力；
3. OpenClaw 能否通过标准接口稳定集成。

如果只解决了第一个问题，后两个没验证，实际跑任务时会频繁失败。

## 做法/步骤

### 1. 硬件评估

先看显存：`nvidia-smi`。没有 N 卡也可以 CPU 推理，但内存建议 32GB 以上，速度会明显下降。家用场景优先考虑 GPU + 量化模型。

### 2. 推理引擎选择

我优先用 Ollama。原因：安装简单、支持 OpenAI 兼容接口、社区模型丰富、CPU/GPU 混合推理友好。如果有 16GB+ 显存且需要高吞吐，可以尝试 vLLM 或 aphrodite，但维护成本更高。

### 3. 模型选择

从 7B/8B 级别开始，例如 Qwen2.5-7B-Instruct 或 LLaMA3.1-8B-Instruct，量化选择 Q5_K_M 或 Q4_K_M。8GB 显存跑 Q4，12GB 跑 Q5/Q6，14B 模型在 16GB 显存下也比较紧张。不要一上来拉 70B。

```bash
ollama pull qwen2.5:7b-instruct-q5_K_M
ollama run qwen2.5:7b-instruct-q5_K_M
```

### 4. 接入 OpenClaw

在 OpenClaw 的模型配置中新增 provider，类型选 OpenAI-compatible：

```yaml
base_url: http://127.0.0.1:11434/v1
model: qwen2.5:7b-instruct-q5_K_M
api_key: ollama
```

Ollama 不校验 key，随便填即可。关键是确认 OpenClaw 请求的是 `chat/completions` 格式，并且支持 `tool_call` 字段。

### 5. 测试工具调用

给 Agent 一个包含 MCP 工具的任务，例如“读取桌面上的 todo.txt 并总结”。观察模型是否输出结构化 function call，而不是自然语言描述。如果失败，先做单工具测试，再逐步增加工具数量。

## 踩坑点

### 显存不足导致 OOM

表现为进程被杀或推理明显变慢。用 `ollama ps` 查看模型层是否 offload 到 GPU，如果 `cpu` 层过多，换成更小量化或缩短上下文。

### 上下文长度不够

Agent 多轮交互很容易超过默认的 4096 或 8192 token。可以在 Modelfile 里设置 `num_ctx` 到 16384 或 32768，但会增加显存，需要权衡。

### 工具调用格式不稳

不少社区模型对 OpenAI function calling 的支持不稳定，容易漏掉 `arguments` 或生成额外文本。建议在 OpenClaw 侧开启 JSON mode，或者先用 native tool calling 做单工具验证。如果不行，就用 prompt 显式描述工具，并配合 `stop` 序列。

### 温度与采样

Agent 任务建议低温度（0.1–0.3）和固定 seed，便于复现。但温度太低可能导致重复输出或死循环，需要实测调整。

### 并发限制

Ollama 默认并发有限，多个 Agent 线程可能排队。可以通过 `OLLAMA_NUM_PARALLEL` 调整，但会增大显存占用。

## 可复用建议

1. **先跑通最小闭环**：本地模型 + 一个简单 MCP 工具，确认请求/响应兼容后再扩展。
2. **准备两个模型**：一个通用对话/规划模型（7B–14B），一个 embedding 模型（如 bge-m3）做本地检索，避免让大模型处理长文档。
3. **用 Modelfile 固化参数**：

```dockerfile
FROM qwen2.5:7b-instruct-q5_K_M
PARAMETER temperature 0.2
PARAMETER num_ctx 16384
PARAMETER stop "<|im_end|>"
```

然后：

```bash
ollama create my-agent-model -f Modelfile
```

4. **监控资源**：`nvtop` 或 `ollama ps` 观察显存，发现频繁 OOM 直接降级。
5. **降低期望**：本地小模型的规划能力有限。在 OpenClaw 里把任务拆小，减少单次推理复杂度，比换模型更有效。

## 总结

在家用电脑上跑本地 LLM 接入 OpenClaw 完全可行，但关键是选对量化、控制上下文、验证工具调用一致性。不要盲目追求大模型，从 7B/8B 起步，把工程链路调通，再按需升级硬件或模型。本地部署的价值不在于替代顶级云模型，而在于低延迟、隐私和可复现的实验环境。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/1ad5ee81f26c8d61.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/b6acea363daab61f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/0cd152fce21bfb1a.png)

