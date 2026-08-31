---
title: 本地 LLM 部署：在家用电脑上跑大模型的实践指南
feedId: 35559
source: 综合讨论
publishedAt: 2026-08-31
---

## 背景

本地部署 LLM 对 OpenClaw/Agent/MCP 用户最大的价值不是“替代大厂 API”，而是获得一个固定版本、可离线、可压测的 OpenAI-compatible 端点。插件调试、MCP 工具循环、自动化脚本跑批时，本地模型能减少限流、数据外发和 token 成本。

## 问题

家用电脑常见瓶颈：显存小、内存带宽有限、模型选型不当、工具调用不稳定。很多人第一次直接下载 70B 或未量化版本，结果 OOM；或者 7B 模型在 Agent 循环里输出 JSON 崩坏，导致 OpenClaw 解析失败。

## 做法/步骤

### 1. 先按硬件选模型，不要贪大

| 可用显存 | 建议模型/量化 |
| --- | --- |
| 8GB | 7B-8B Q4_K_M |
| 16GB | 13B-14B Q4/Q5 |
| 24GB | 32B Q4 或 14B Q8 |
| 无独显，32GB RAM | 7B-13B Q4 纯 CPU，降低并发 |

优先选 `-instruct` 版本，不要用 base 模型做 Agent。Qwen2.5、Llama 3.1/3.2、Mistral 系列对工具调用和 JSON 输出相对更稳。

### 2. 用 Ollama 部署为本地端点

```bash
ollama pull qwen2.5:7b-instruct-q4_K_M
OLLAMA_HOST=127.0.0.1:11434 OLLAMA_NUM_PARALLEL=1 OLLAMA_KEEP_ALIVE=10m ollama serve
```

固定 `OLLAMA_NUM_PARALLEL=1` 对家用 GPU 更现实，避免并发推理把显存打爆。

### 3. 通过 Modelfile 约束输出

如果模型模板不对，或 Agent 循环停不下来，可以新建 Modelfile：

```text
FROM qwen2.5:7b-instruct-q4_K_M
PARAMETER temperature 0.0
PARAMETER num_ctx 8192
PARAMETER stop "Observation:"
```

然后 `ollama create local-agent -f Modelfile`。`num_ctx` 至少 8192，MCP 多轮很容易爆上下文。

### 4. 接入 OpenClaw 或 MCP

Ollama 提供 OpenAI-compatible 接口：

```bash
curl http://127.0.0.1:11434/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"local-agent","messages":[{"role":"user","content":"reply json only"}],"temperature":0}'
```

在 OpenClaw 的模型配置里使用同样参数：`base_url` 指向 `http://127.0.0.1:11434/v1`，`api_key` 可填 `ollama`，模型名填 `local-agent`。

### 5. Agent/MCP 工具调用建议

- MCP 工具描述保持扁平，不要嵌套太深，`required` 字段清晰。
- 温度设为 0.0，关闭随机采样。
- 如果 native tool_calls 不稳定，让模型输出 JSON，由 OpenClaw 插件层解析，再调用 MCP server。这比强依赖模型的 function calling 更可控。

## 踩坑点

1. **Prompt template 不匹配**  
   第三方 GGUF 导入后，Ollama 自动模板可能错。用 `ollama show --modelfile` 检查，必要时手动指定 TEMPLATE。

2. **Context 默认太小**  
   API 不传 options 时可能是 2048/4096。长任务失败先怀疑上下文长度，而不是模型笨。

3. **工具调用非原生**  
   小模型输出 JSON 可能出现尾随逗号、缺失括号。用 `format: json` 或 `response_format`，并做后处理容错。

4. **Windows/WSL2**  
   有 N 卡优先用 Windows 原生 Ollama；WSL2 里 GPU 透传和显存分配问题更多。

5. **网络暴露风险**  
   绑定 `0.0.0.0` 仅用于局域网调试，Windows 防火墙不要直接全开。

## 可复用建议

- 把 Modelfile、启动脚本、curl smoke test 放进仓库，固定模型 digest。
- 本地模型只做结构化抽取、路由、简单工具选择；复杂推理回退到更大模型。
- 每次升级模型后跑一组结构化输出用例，记录通过率。
- 用 `ollama ps` / `nvidia-smi` 看显存和 KV cache 占用。
- 给 OpenClaw 留 `local` 和 `cloud` 两个 profile，避免改全局配置。

## 总结

本地 LLM 部署不是一次性下载模型，而是把“模型端点”当作工程组件管理：固定量化版本、限定上下文、约束输出、测试工具调用。对 OpenClaw/Agent/MCP 用户，本地端点在离线调试、插件开发和批量结构化任务里非常实用，但复杂推理要控制预期。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/a743b20792e5f75a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/8e78d75edc55a9a4.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/f1304f7b458dc946.png)

