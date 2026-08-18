---
title: 本地 LLM 部署：家用电脑跑 7B–14B 模型并接入 OpenClaw 的工程化清单
feedId: 33718
source: 综合讨论
publishedAt: 2026-08-18
---

# 本地 LLM 部署：家用电脑跑 7B–14B 模型并接入 OpenClaw 的工程化清单

## 背景

很多 OpenClaw/Agent/MCP 用户在跑自动化任务时，默认接云端大模型 API。但一旦涉及内网数据、离线环境、API 限流或需要固定版本的调试场景，本地 LLM 就变成一项刚需。它不追求超越 GPT-4，而是提供一个确定、可复现、低成本的执行器。

本文不聊“本地模型替代云端”的伪命题，只给一套能在普通家用电脑上落地的部署方案。

## 要解决的三个问题

家用电脑跑本地 LLM，核心约束只有三个：

1. **显存/内存不够**：7B 以上模型 FP16 权重会爆显存，必须量化。
2. **推理速度**：CPU 跑 7B Q4 约 5–10 tok/s，能接受但偏慢；有独显会好很多。
3. **工具调用能力**：7B/14B 模型对复杂 function calling 或 MCP schema 的遵循度明显弱于大模型，需要做约束和简化。

## 部署步骤

### 1. 硬件与模型选择

按可用显存/内存选模型，别贪大：

| 硬件条件 | 推荐模型规模 | 推荐量化 |
|---|---|---|
| 16GB 内存，无独显 | 7B | Q4_K_M |
| 8GB 显存 | 7B/8B | Q4_K_M |
| 12GB 显存 | 14B | Q4_K_M |
| 24GB 显存 | 32B | Q4_K_M 或 14B Q8_0 |

常用选择：`Qwen2.5-7B-Instruct`、`Llama-3.1-8B-Instruct`、`Mistral-Nemo-12B-Instruct`。优先选指令微调版本，裸基座模型在工具调用场景很难用。

### 2. 推理引擎

家用场景推荐 **Ollama**，安装成本最低，默认提供 OpenAI 兼容 API。如果你需要更细的参数控制，可以换 `llama.cpp` 或 `LM Studio`，但 Ollama 对 OpenClaw 接入最友好。

启动时建议显式设置上下文长度和并发，避免默认值把显存打爆：

```bash
OLLAMA_CONTEXT_LENGTH=8192 OLLAMA_NUM_PARALLEL=1 ollama serve
```

然后拉取模型：

```bash
ollama pull qwen2.5:7b-instruct-q4_K_M
```

### 3. 接入 OpenClaw/Agent

Ollama 默认 OpenAI 兼容端点在 `http://127.0.0.1:11434/v1`。OpenClaw 配置里把 LLM 地址指过去即可，API key 随便填，但要保持模型名与本地一致。

例如：

```yaml
llm:
  base_url: http://127.0.0.1:11434/v1
  api_key: ollama
  model: qwen2.5:7b-instruct-q4_K_M
```

### 4. 工具调用验证

先跑一个两步工具调用测试：让模型先查询一个 MCP 工具的参数，再根据返回结果执行动作。重点看参数提取是否准确、是否提前截断、是否遵守 JSON 输出。小模型很容易在第二步“脑补”结果，需要额外校验。

## 踩坑点

- **上下文长度不是越大越好**：设成 8192 时 KV cache 会显著增加显存占用。8GB 显存跑 7B 如果设 32K 上下文很容易 OOM。建议从 4096 起调。
- **小模型 function calling 不稳定**：工具数量控制在 3 个以内，schema 尽量扁平，参数名用短单词，避免嵌套对象。
- **Windows 下显存被占用**：浏览器、游戏、桌面硬件加速都会抢显存。跑模型前先关掉浏览器 GPU 加速，或直接重启 Ollama。
- **输出重复/停不下来**：设 `temperature=0.1`、`repeat_penalty=1.1`，并在 system prompt 中明确停止条件。Ollama 的 `stop` 参数也建议加上。
- **不要用 Q8 以上量化**：在家用卡上 Q8 或 FP16 会直接爆显存。Q4_K_M 是速度和效果的平衡点，Q5_K_M 稍慢但略稳。
- **MCP 工具 schema 太复杂**：7B 模型经常把嵌套 JSON 参数解析错。可以写一层“简化包装”，把复杂 MCP 工具封装成参数简单的小工具给本地模型调用。

## 可复用建议

1. **用 Modelfile 固化配置**：把 system prompt、temperature、stop tokens、上下文长度写进 Modelfile，生成固定 tag。这样每次启动行为一致，方便调试和回滚。
2. **写健康检查脚本**：定期请求 `/api/tags` 和 `/v1/models`，确认模型已加载、显存未溢出。
3. **本地模型定位为执行器**：简单、隐私、离线的任务交给本地模型；复杂推理、长链条规划仍用云端兜底。两者可以共存。
4. **固定模型版本**：记录 `ollama list` 里的 digest 或 tag，避免自动更新导致行为变化。
5. **限制 JSON 输出**：如果引擎支持 grammar/json mode，优先开启。否则在 prompt 里强制要求“只输出 JSON，不要解释”。

## 总结

家用电脑部署本地 LLM 跑 OpenClaw 自动化，是可行的，但前提是放弃“小模型平替大模型”的幻想。把上下文、量化、工具数量和输出格式都约束住，7B–14B 模型能稳定处理不少隐私、离线、低成本任务。它的价值不是更强，而是**可控、可复现、不依赖外部 API**。管理好预期，本地 LLM 就是 OpenClaw 栈里一个很好用的执行单元。

---

