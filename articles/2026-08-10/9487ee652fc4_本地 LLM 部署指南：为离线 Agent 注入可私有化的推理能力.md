---
title: 本地 LLM 部署指南：为离线 Agent 注入可私有化的推理能力
feedId: 32384
source: 综合讨论
publishedAt: 2026-08-10
---

## 背景：为什么需要在本地跑大模型

当你开始用 OpenClaw 构建自主 Agent、编写 MCP 插件或编排自动化流程时，很快会碰到三个现实约束：

1. **延迟**：每次工具调用、反思、规划都要走云端 API，网络抖动会让 Agent 循环慢得不可控。
2. **成本**：复杂任务可能产生上百次推理调用，即便使用小参数模型，账单也会指数级上升。
3. **隐私与合规**：某些执行环境（如内网运维、本地文件处理）不允许将上下文外传。

在这些场景下，把 LLM 部署到本地就成了一条务实的替代路径。本文不讨论“取代 GPT-4”之类的野心，只聚焦一个目标：让家里的台式机或工作站稳定地为 OpenClaw 这类自动化框架提供可控、低延迟、零外部依赖的推理后端。

## 问题拆解：本地推理的硬伤

在家用电脑上跑大模型，面临两个核心挑战：

- **硬件资源受限**：多数开发者的显卡在 8-12 GB 显存之间，能跑多大的模型？量化后效果如何？
- **工具调用能力缺失**：Agent 依赖 function calling，而多数本地开源模型对结构化输出的遵从性远弱于闭源模型，容易产生“幻觉参数”或格式错误。

这意味着，本地部署不是“下载即用”，需要围绕模型选择、推理配置、输出约束做一层工程适配。

## 实践步骤：从 0 到 1 接入 OpenClaw

### 1. 环境与模型选型
推荐使用 [Ollama](https://ollama.com) 作为运行后端，它屏蔽掉了 llama.cpp 的大部分编译细节，并内建了模型管理和 OpenAI 兼容 API。

- **显卡要求**：推荐至少 8GB 显存的 NVIDIA GPU（2060/3060/4060 均可），纯 CPU 推理也可以跑 7B 级，但延迟会高到失去实用价值。
- **模型选择**：优先选择已明确支持 function calling 的 7B-8B 参数模型，例如 Llama 3.1 8B Instruct、Qwen2.5 7B Instruct、Mistral 7B Instruct v0.3。这类模型在量化后仍能保留基本的工具调用能力。

安装并启动：
```bash
curl -fsSL https://ollama.com/install.sh | sh
ollama serve
```

拉取模型（以 Qwen2.5 7B 4bit 量化版为例）：
```bash
ollama pull qwen2.5:7b-instruct-q4_K_M
```

验证推理：
```bash
curl http://localhost:11434/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"qwen2.5:7b-instruct-q4_K_M","messages":[{"role":"user","content":"你好"}]}'
```

### 2. 与 OpenClaw 集成
OpenClaw 的 LLM 后端配置通常位于 `config.yaml` 或环境变量中。将 provider 指定为 `ollama` 并填入本地模型名：

```yaml
llm:
  provider: ollama
  base_url: http://localhost:11434/v1
  model: qwen2.5:7b-instruct-q4_K_M
  temperature: 0.1
  max_tokens: 2048
```

重新启动 OpenClaw 后，所有 Agent、MCP 插件的推理请求都会路由到本地。

### 3. 让 function calling 可靠运行
这是本地部署失败率最高的环节。开源模型经常输出缺失字段、错误 JSON 或自行补充不存在的函数名。需要加三层保护：

- **在 system prompt 中明确定义输出格式**，并给出完整的 JSON Schema 示例。
- **使用 Ollama 的 `format: json` 参数强制 JSON 模式**（需要在 Modelfile 中设置或通过 API 传入）。
- **在 Agent 层加一层轻量校验**：捕获 JSON 解析错误时重试一次，同时将错误信息拼回上下文，让模型纠正。

实测中，为 Qwen2.5 7B 添加一个 200 行左右的 system prompt 模板，即可将工具调用成功率从 60% 提升到 90% 以上。

## 踩坑记录

1. **显存溢出（OOM）**
   即使模型文件只有 5GB，推理时的 KV cache 和上下文窗口会额外吃掉大量显存。如果使用 8K 以上上下文，8GB 显存几乎会爆。解决方法：在 Modelfile 中限制 `num_ctx` 为 4096，或使用更低量化版本（如 IQ2_M）。

2. **模型不支持原生 tool calling**
   部分模型的 Instruct 版本未训练过 tool 格式，此时 direct prompt engineering 无解。切换为经过 tool-use 微调的版本（如 `qwen2.5:7b-instruct`）是唯一办法。

3. **并发导致性能骤降**
   家用 GPU 通常只能串行处理，同时来 5 个推理请求会直接排队到不可用。可在 OpenClaw 侧配置 `max_concurrent_requests: 1`，或引入本地代理（如 LiteLLM）做请求排队与超时控制。

4. **温度与输出长度的微妙关系**
   为 Agent 设 temperature 为 0 并不总是最优，部分模型会在 0 温度下陷入重复循环。建议设为 0.1～0.2，配合 `repeat_penalty=1.05` 来抑制重复。

## 可复用建议

- **构建设备-场景矩阵文档**：记录下你的硬件在哪些任务上可用（例如“8GB VRAM + Qwen2.5 7B，能跑代码审查 Agent，但无法支撑长文档总结”），未来扩展 Agent 时直接查表匹配。
- **使用 Grammar 约束替代 JSON 模式**：Ollama 支持 GBNF 语法，可以精确定义输出的 JSON 结构，比 format: json 更可靠。
- **为本地模型做健康检查**：在 OpenClaw 启动时增加一个探活脚本，发送固定 tool-call 请求并验证返回格式，失败则回退到云端模型或报警。
- **保留一条云端 fallback 管线**：当检测到本地连续 3 次解析错误或超过 10 秒未返回时，自动切换到远程 API，保障自动化流程不中断。

## 总结

在家用电脑上部署 LLM 为 OpenClaw Agent 提供推理，是一件完全可以落地的工程优化。它不会被标榜为“替代 GPT”的银弹，但在特定约束下（隐私、低延迟、零成本迭代）却是最务实的方案。核心工作在于：选择对工具调用友好的 7B 模型，精准控制量化与上下文窗口，以及在 Agent 层加入防御性结构化解析。一旦这套栈稳定下来，你会发现日常的自动化开发、测试和调试不再需要保持公网连接，整个 Agent 系统的可观测性和可控性也会上升一个台阶。

---

