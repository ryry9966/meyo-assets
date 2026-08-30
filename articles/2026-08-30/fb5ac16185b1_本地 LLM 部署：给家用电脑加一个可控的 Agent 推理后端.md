---
title: 本地 LLM 部署：给家用电脑加一个可控的 Agent 推理后端
feedId: 35323
source: 综合讨论
publishedAt: 2026-08-30
---

## 背景

在 OpenClaw、Agent、MCP、插件自动化的工程实践里，很多问题不是框架逻辑不对，而是模型接口不稳定、调用限频、数据不能出内网，或者调试工具调用时没有可复现的返回。家用电脑本地跑一个 LLM，可以充当固定、可审计、免额度的推理后端，适合工具调用调试、结构化提取、低风险自动化和敏感数据任务。

但它不是云模型的替代品。家用 GPU 通常只有 8GB、12GB 或 16GB 显存，能用的是 7B-14B 级别的量化模型。目标应当是把本地 LLM 当成一个“可控组件”，而不是“完整大脑”。

## 问题

在接入 OpenClaw/Agent/MCP 前，常见问题有四个：

1. 显存不够，模型层无法全部 offload 到 GPU，CPU 推理极慢。
2. 本地小模型工具调用能力弱，容易输出非法 JSON、漏调用、重复调用。
3. Agent/MCP 注入的 system prompt、工具 schema、历史消息会把上下文迅速塞满。
4. OpenAI 兼容接口不完整，流式响应、finish_reason、usage 字段可能与云端不一致。

## 做法/步骤

### 1. 选推理栈

家用场景优先用 Ollama，安装简单、自带 OpenAI 兼容 API。需要更细控制时用 llama.cpp server。不要直接上 vLLM，它对家用单卡和量化模型并不是首选项。

### 2. 选模型与量化

选择 7B-14B 的 instruct 模型，Q4_K_M 或 Q5_K_M 量化。先用 8k-16k 上下文，显存稳定后再考虑 32k。不要一上来就拉最大上下文，显存和速度代价都很高。

```bash
ollama pull qwen2.5:14b-instruct-q4_K_M
ollama create local-agent -f Modelfile
```

Modelfile 示例：

```text
FROM qwen2.5:14b-instruct-q4_K_M
PARAMETER num_ctx 16384
PARAMETER temperature 0.1
PARAMETER stop "<|im_end|>"
```

### 3. 启动与接入

```bash
ollama serve
export OPENAI_BASE_URL=http://127.0.0.1:11434/v1
```

在 OpenClaw/Agent 模型配置里使用：

```text
base_url: http://127.0.0.1:11434/v1
api_key: dummy
model: local-agent
```

建议先让本地模型只做单步工具选择或结构化提取，不要直接承担复杂多步规划。

### 4. MCP 工具调用约束

如果模型输出工具调用不稳定，不要把 20 个 MCP 工具全丢进去。只保留 3-5 个关键工具，描述压缩到最短，要求模型输出固定格式：

```json
{"tool":"search","args":{"q":"..."}}
```

然后在客户端代码里做 schema 校验，失败时重试一次。Ollama 的 `format: json` 只是 JSON 约束，不等于完整 schema 校验，不能完全依赖。

## 踩坑点

- **显存不释放**：Ollama 默认可能保留模型在显存中。设置 `OLLAMA_KEEP_ALIVE=5m`，避免占着显存影响其他任务。
- **上下文爆掉**：Agent/MCP 工具 schema 很长，16k 很容易不够。先裁剪工具定义和历史消息，再考虑升上下文。
- **推理速度**：本地小模型低于 8-10 tok/s 时，交互式 Agent 体验会明显变差。可以用 `ollama run --verbose` 查看 eval token/s。
- **提示词模板不匹配**：不同模型对 system/stop 模板要求不同，模板错误会导致工具调用时好时坏。
- **流式兼容问题**：调试期优先用 `stream=false`，等工具链路稳定后再开流式。

## 可复用建议

把本地推理栈固化为可复现配置：

- Modelfile、启动参数、JSON schema 放入 git。
- 用 docker compose 或 systemd 管理 `ollama serve`。
- 记录基准测试：模型、量化、上下文长度、tok/s、显存占用。
- 本地 LLM 只负责边界清晰的任务：敏感数据抽取、离线 tool call mock、回归测试、固定格式输出。
- 复杂规划、长链条工具调用仍交给云端大模型。

## 总结

家用电脑本地部署 LLM，不是为了在本地复刻一个 GPT-4，而是为 OpenClaw/Agent/MCP 链路增加一个可控、可重复、可离线验证的推理后端。把模型选型、量化、上下文、工具约束、接口配置固定下来，比频繁换更大模型更有效。少一点模型崇拜，多一点工程约束，本地 LLM 才能真正进入自动化流水线。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/06a4fb8a488785d6.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/679dbfb3f8a77e43.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/5a64a69eb4e4e7ae.png)

