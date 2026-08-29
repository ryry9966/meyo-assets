---
title: 本地 LLM 部署：在家用电脑上跑大模型的实践指南
feedId: 35317
source: 综合讨论
publishedAt: 2026-08-30
---

## 背景
对 OpenClaw/Agent/MCP/插件/自动化玩家来说，本地 LLM 的价值不是“隐私叙事”，而是可重复调试、无速率限制、无 token 账单，适合做 agent 工具调用回归。代价是家用电脑算力有限，7B/14B 级别模型能跑，但“能聊天”和“能稳定调用工具”是两回事。

## 问题
家用 GPU 常见 8GB/12GB/16GB VRAM，或者只有 CPU+RAM。Agent 场景里，system prompt、工具 schema、历史消息、MCP 返回结果会迅速占用上下文。本地推理还要处理 prompt 处理速度、并发限制、量化损失。关键不是能加载模型，而是 tool call JSON 是否稳定。

## 做法
### 1. 先按显存选模型，而不是按榜单
- 8GB VRAM：7B–8B Q4_K_M，上下文 4096–8192。
- 12–16GB VRAM：14B Q4_K_M，或 7B–8B Q5/Q6/Q8，上下文 8192。
- 32GB 以上或统一内存：可尝试 30B–32B Q4_K_M，但 prompt 处理可能仍慢。

原则：给 KV cache 留 1.5–2GB；不要用 Q2/Q3 跑工具调用。

### 2. 推理栈选型
- 快速接入：Ollama，提供 OpenAI 兼容接口 `/v1`，适合 OpenClaw/MCP 直接挂。
- 需要精细控制：llama.cpp + Modelfile/启动参数，方便调 `--ctx-size`、`--threads`、`--gpu-layers`。
- 高吞吐或多实例：vLLM，但家用单卡优势不大。

建议先用 Ollama 跑通，再换 llama.cpp 压榨参数。

### 3. 模型选择与模板
选择明确标注支持 function calling 的 instruct 模型，例如 Qwen 系列、Llama 3.x、Mistral 等。不要只看通用跑分。用 Ollama 时确认 `TEMPLATE` 和 `SYSTEM` 与模型原生 chat template 一致，否则工具调用会泄漏特殊 token。

### 4. 接入 OpenClaw/MCP
设置 base_url 为 `http://127.0.0.1:11434/v1`，API key 随意填，model 填 Ollama 模型名。示例：

```yaml
base_url: http://127.0.0.1:11434/v1
api_key: local
model: qwen2.5:7b-instruct-q4_K_M
```

MCP 工具描述尽量扁平化：减少工具数量、缩短 description、避免深层嵌套 schema。每个工具定义都可能占 200–800 tokens。

### 5. 验证工具调用
用固定用例测试，例如“调用 weather(city='Beijing')”，连续跑 20 次，检查 JSON 可解析率、参数正确率、是否夹带解释文本。工具调用温度设 0.1–0.2，关闭花哨采样。

## 踩坑点
- Q4 以下量化在 tool call 上容易出现残缺 JSON 或错参数名；至少 Q4_K_M，优先 Q5_K_S/Q6_K。
- CPU offload 后 prompt 处理很慢，尤其上下文超过 8192。可减少 MCP 返回内容、截断工具输出，或用较小模型处理长上下文。
- 多 agent/并发请求会让家用 GPU 排队，出现超时。本地模型最好一次只跑一条链路。
- MCP 返回大量 HTML/日志会快速填满 context，造成循环失败。在工具侧做摘要或字段过滤。
- 忽略 prompt template 导致模型输出 `<|tool_call|>` 等标记，OpenClaw 无法解析。先用 `ollama run` 和 verbose 日志检查原始输出。

## 可复用建议
- 建一个本地 tool call 回归集，每次换模型或量化版本跑一遍。
- 把 GGUF 文件名、量化、上下文大小、工具 schema 版本记录下来，方便复现。
- 使用 grammar/structured output 约束 JSON，能显著降低解析失败。
- 监控 VRAM、单 token 速度、prompt eval 时间；不要只盯着“能跑”。
- 先跑 8B Q4 作为基线，再逐步升级到 14B；不要一上来追 32B。

## 总结
本地 LLM 部署的难点不是下载模型，而是让它在 agent/MCP 场景里稳定地完成工具调用。控制上下文、量化、工具 schema 和并发，比堆参数更实际。先把小模型跑稳，再扩展规模。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/b919078538e87916.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/2dd9818e5d4719a2.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/45d029f513714a54.png)

