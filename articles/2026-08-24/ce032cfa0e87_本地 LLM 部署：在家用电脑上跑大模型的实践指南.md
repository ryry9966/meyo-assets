---
title: 本地 LLM 部署：在家用电脑上跑大模型的实践指南
feedId: 34496
source: 综合讨论
publishedAt: 2026-08-24
---

# 本地 LLM 部署：在家用电脑上跑大模型的实践指南

## 背景

在 OpenClaw、Agent、MCP、插件与自动化实践里，模型调用通常是最大外部依赖。云端 API 有延迟、限流、费用和数据出境问题；很多自动化任务只是“读工具输出、总结状态、格式化回复”，不需要最强模型。家用电脑如果有 16GB 以上内存、8GB 以上显存，已经可以稳定跑 7B-14B 量化模型，处理本地 Agent 的轻量推理。

## 问题

本地部署不是“下载模型就能用”，常见问题包括：

- 显存/内存不足，OOM 或爆 KV cache；
- 模型不会调用工具，导致 OpenClaw 的 MCP/插件链路失败；
- OpenAI 兼容层不完整，部分字段缺失；
- 模板与停止符不对，输出停不下来。

## 做法/步骤

### 1. 选择推理后端

优先 Ollama，安装简单、API 兼容好、模型管理省心。如果需要更细粒度控制，可用 llama.cpp 或 LM Studio。下述以 Ollama 为例。

### 2. 选择模型和量化

建议从以下级别开始，而不是直接上最大模型：

- 8GB 显存：Qwen2.5-7B-Instruct 的 Q4_K_M，或 Llama-3.1-8B-Instruct Q4_K_M
- 12GB 显存：Qwen2.5-14B-Instruct Q4_K_M
- 纯 CPU / 32GB 内存：Qwen2.5-7B-Instruct Q4_0，速度可接受但不要期待交互式高速

需要工具调用的场景，优先选择明确支持 function calling 的 Instruct 版本，不要用 base 模型或随机社区微调模型。

### 3. 启动并验证 API

```bash
ollama pull qwen2.5:7b-instruct-q4_K_M
ollama serve
# 另开终端
curl http://localhost:11434/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "qwen2.5:7b-instruct-q4_K_M",
    "messages": [{"role": "user", "content": "ping"}]
  }'
```

确认返回 JSON 正常，且 `finish_reason` 不是一直 `length`。

### 4. 接入 OpenClaw / Agent

OpenClaw 通常允许自定义 OpenAI-compatible base URL。将：

- base URL 设置为 `http://localhost:11434/v1`
- API key 填任意非空值，如 `local`
- model 填 `qwen2.5:7b-instruct-q4_K_M`

然后启动一个最小 Agent 测试：

- 不带工具：先跑一句总结/分类，确认对话模板正确；
- 再带一个 MCP 工具或本地插件：让模型读取文件/查询状态，验证工具调用 JSON 是否被正确触发。

### 5. 固定参数，避免默认值干扰

本地模型对采样参数敏感。建议显式设置：

- `temperature=0.1` 或 `0.2`（Agent/工具调用不要超过 0.3）
- `top_p=0.9`
- `num_ctx=4096` 或根据任务需要设置，先小后大
- 停止符保持后端默认，不要随意加 `<|im_end|>` 以外的自定义停止符

## 踩坑点

1. **显存不足但没报 OOM**：Ollama 会部分 offload 到 CPU，速度突然下降。需要看日志里的 `offload` 层数。若 GPU offload 不到 100%，要么换小模型，要么降低上下文。
2. **工具调用不稳定**：7B/8B 模型可以调工具，但比云端大模型更容易生成错误 JSON。需要：
   - 在系统提示中给出 1 个工具调用示例；
   - 让工具名和参数简短；
   - OpenClaw 侧增加 JSON 解析失败重试。
3. **长上下文爆 KV cache**：不要默认给 Agent 塞入 32k 上下文。`num_ctx` 不是配置越高越好，会线性增加显存占用和推理时间。先固定到 4096/8192，跑通后再上调。
4. **模型输出停不下来**：常见原因是模板不匹配。用 Ollama 的模型名要准确带 `instruct`，不要自己改 prompt 模板，尤其不要给对话模型加 `### Human/Assistant` 这类旧模板。
5. **认为本地模型能替代所有云端模型**：本地适合状态汇总、格式化、工具编排、小规模 RAG、离线任务；复杂推理、长文档规划仍应回退云端或使用更大模型。

## 可复用建议

- **先把最小闭环跑通**：本地 API → OpenClaw 无工具对话 → 一个 MCP 工具。不要一上来接 10 个插件。
- **维护一个稳定模型组合**：例如 `qwen2.5:7b-instruct-q4_K_M` 作为日常 Agent 模型，`qwen2.5:14b-instruct-q4_K_M` 作为复杂任务模型。
- **监控指标**：重点看首 token 延迟、每秒 token 数、GPU offload 层数、上下文占用。可写一个小脚本定期请求 `/api/ps` 查看显存。
- **保留回退策略**：OpenClaw/Agent 配置中可以按任务复杂度或错误率回退到云端 API，本地模型作为默认执行层。
- **模型文件版本固定**：不要频繁升级模型 tag。`latest` 会漂移，使用精确 tag 如 `qwen2.5:7b-instruct-q4_K_M`，需要升级时先在测试环境验证。

## 总结

把本地 LLM 接进 OpenClaw/Agent/MCP/插件体系，第一步不是追求“完全离线替代”，而是找到适合本地的轻推理任务：状态汇总、工具编排、格式化输出、失败重试。用 Ollama 固定一个 7B-14B 的 Instruct 量化模型，设置低温度、小上下文，先把单工具闭环跑稳，再逐步扩展。控制上下文、显存和工具调用复杂度，比单纯换大模型更能提高稳定性。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/f9e24b463abe7187.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/d4358eb26e4517fd.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/853e852ac425f216.png)

