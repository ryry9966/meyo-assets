---
title: 本地 LLM 部署：家用电脑跑大模型的工程化实践
feedId: 34605
source: 综合讨论
publishedAt: 2026-08-25
---

## 背景

给 OpenClaw、Agent、MCP 插件和自动化流程接本地 LLM，主要动机不是“我要本地部署”，而是：某些任务不想出内网、token 成本需要控制、调试时希望模型接口稳定可控。家用电脑跑大模型，最现实的问题不是“能不能跑”，而是“跑起来之后工具调用是否可用、延迟是否可接受、内存/显存是否会被打爆”。

这篇不讨论“一人一卡跑千亿模型”的幻想，只给一套能复现的本地推理栈。

## 问题

普通家用电脑的瓶颈通常是显存或内存带宽。一个 7B/8B 模型做 Q4 量化后，大约需要 6–8GB 显存；14B Q4 约 10–12GB；32B Q4 约 20GB。如果你用 CPU 推理，32B Q4 可能勉强加载，但 tokens/s 往往只有个位数，用在 Agent 多轮工具调用里会很痛苦。

另一个更隐蔽的问题是：能对话不代表能稳定调用工具。Agent/MCP 依赖 function calling 或 JSON 输出，小模型经常漏参数、多输出解释文字、把工具名写错。因此本地部署不能只选“会聊天”的模型，还要选指令遵循和工具调用表现稳定的 instruct 模型。

## 做法/步骤

### 1. 先盘点资源，再选模型

建议从 7B/8B Q4_K_M 开始。别一上来就拉 32B，否则加载完可能只剩 1–2GB 给 KV cache，多轮对话很快超上下文。目标显存占用建议控制在物理显存的 70% 以内，留出 KV cache 和推理峰值。

### 2. 推理栈选 Ollama，接口统一成 OpenAI 兼容

家用场景优先用 Ollama，原因很简单：启动快、GGUF 支持好、自带 OpenAI 兼容接口。OpenClaw 或多数 Agent 框架都默认支持 `base_url` 指向 `http://localhost:11434/v1`，`api_key` 可填任意非空字符串。llama.cpp 可作为更细粒度控制的后备，vLLM 在家用单卡上收益不高。

### 3. 下载模型并校验

优先从 Ollama library 或 HuggingFace 下载 GGUF。下载后记录文件大小，必要时用 sha256 校验。模型文件建议放在独立数据盘，避免占用系统盘。

### 4. 用 Modelfile 固定推理参数

不要用默认参数直接跑 Agent。示例 Modelfile：

```bash
FROM ./qwen2.5-7b-instruct-q4_k_m.gguf
PARAMETER temperature 0.1
PARAMETER num_ctx 8192
PARAMETER top_p 0.9
PARAMETER stop "<|im_end|>"
```

对工具调用任务，`temperature` 建议 0.1–0.3。`num_ctx` 不要一上来就给 32k，先跑 8k，确认稳定后再调整。`stop` 参数要匹配模型模板，否则多轮工具调用容易输出不完整。

### 5. 启动服务并做健康检查

```bash
ollama serve
```

启动后先请求 `/v1/models` 确认模型名一致，再发一个最小 `chat/completions` 请求验证。不要直接接到 OpenClaw 里盲目跑任务。

### 6. 接入 OpenClaw/MCP

配置模型服务地址时，模型名要与 `ollama list` 中的名称完全一致。MCP 工具接入后，先做单轮工具调用测试，再做多轮工具调用测试。观察是否有工具名错误、参数缺失、多余解释文字。

## 踩坑点

- **显存被 KV cache 占满**：`num_ctx` 过大时，推理初期没事，多轮对话后突然 OOM。先设置保守上下文长度。
- **工具调用输出带解释文字**：小模型常见。降低 temperature、限制输出格式、使用 JSON schema 会有改善，但不能完全根治。
- **OpenAI 兼容层不完整**：部分模型模板和 Ollama 兼容层配合不好，导致工具调用字段丢失。换模型或自定义 Modelfile 模板通常能解决。
- **并发过高**：家用单卡不要挂太多并发请求，避免排队超时。可以在 Agent 侧设置单请求队列，或限制 max concurrency。
- **模型下载中断**：用支持断点续传的方式下载，避免损坏后加载时报错。
- **首次加载慢**：模型冷启动可能需要几十秒到几分钟。设置 `OLLAMA_KEEP_ALIVE` 保持驻留，避免每次请求都重新加载。

## 可复用建议

1. **建立本地模型基线**：记录 tokens/s、显存占用、单轮工具调用成功率、多轮工具调用成功率，换模型后可以对比。
2. **独立部署模型服务**：Ollama 作为独立进程运行，不要嵌在 Agent 主进程里。重启 Agent 时模型可以保持驻留。
3. **工具描述尽量短**：给本地小模型提供少而清晰的工具描述，工具数量不要一次给太多。
4. **做好回退路径**：本地模型不可用时，回退到远程小模型或简单规则，避免整个自动化流程中断。
5. **监控显存和延迟**：至少记录一次峰值显存和平均首字延迟，方便判断能否扩上下文或换更大模型。

## 总结

家用电脑跑本地 LLM，真正难的不是下载模型，而是让模型在 Agent/MCP 环境里稳定输出工具调用。工程上的关键动作是：先小后大、固定参数、统一接口、做健康检查、限制并发、准备回退。做到这些，本地 LLM 可以成为一条可用的内网推理通道，而不是一个只有截图价值的玩具。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/cc16e46934659fc9.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/dea09ec0c4d001ed.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/eee3443be7666d69.png)

