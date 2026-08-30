---
title: 本地 LLM 部署：在家用电脑上跑通可被 Agent 调用的模型
feedId: 35391
source: 综合讨论
publishedAt: 2026-08-30
---

## 背景
在 OpenClaw/Agent/MCP 这些自动化链路里，模型常常是云端 API。但有些场景需要离线、低延迟或处理本地数据。家用电脑跑本地 LLM 不是新问题，真正要解决的不是“能不能跑”，而是“能不能被 Agent 稳定调用”。

如果只是聊天，装个 Ollama 就能跑；但如果要接插件、工具调用和 MCP，就需要按工程化标准做约束。

## 问题
家用 GPU 显存有限，小模型工具调用不稳定，上下文开大后容易 OOM，API 兼容层也有差异。实际目标应定为：在固定硬件下，跑一个可重复、可被本地 Agent 调用的 OpenAI 兼容端点，首 token 延迟可接受，输出 schema 稳定。

## 做法
1. 先估算显存/内存。7B Q4_K_M 权重约 4.2GB，加上 KV cache 和输入输出 buffer，8GB 显存建议 context 不超过 8192。无独显时用 CPU 跑 7B Q4，速度约 5–15 tok/s，只适合轻量任务。

2. 用 Ollama 部署：
```bash
ollama pull qwen2.5:7b-instruct-q4_K_M
```
也可以选 `llama3.1:8b-instruct-q4_K_M`。必须选 instruct 版，base 版不会按工具格式输出。

3. 启动后测 OpenAI 兼容端点：
```bash
curl http://localhost:11434/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"qwen2.5:7b-instruct-q4_K_M","messages":[{"role":"user","content":"ping"}]}'
```

4. 接入 OpenClaw 或自建 Agent 时，把 `base_url` 指向 `http://127.0.0.1:11434/v1`，`api_key` 填任意字符串。不要用 `/api/chat` 原生端点，除非你只做聊天。

5. 涉及工具调用/MCP，优先让模型输出 JSON，而不是指望小模型完美调用原生 function calling。系统提示里写清可用工具、参数类型和输出格式；Ollama 对部分模型支持 `format: "json"`，可以强制 JSON 输出。

6. 固化配置：Ollama 设置 `OLLAMA_NUM_PARALLEL=1`，避免并发抢占显存；context 通过 API 参数或 Modelfile 设定，不要默认开很大。

## 踩坑点
- **OOM**：最常见。显存不够时先降 context，再降量化，不要同时加载两个模型。`nvidia-smi` 看显存，`ollama ps` 看当前模型。
- **工具调用不稳定**：7B 模型经常漏参数、多输出解释文本。解决方式是减少工具数量、缩短参数名、强制 JSON，或换 14B 以上模型。
- **API 兼容差异**：Ollama 的 `/v1` 基本兼容，但 `tool_choice`、`response_format` 等高级字段可能被忽略或表现不一致。接 Agent 时建议先走“普通 chat + 解析 JSON”，不要一上来依赖原生 function calling。
- **模板错误**：自定义 GGUF 导入时，如果 prompt template 不对，输出会混乱。优先用 Ollama 官方库模型，或写好 Modelfile。

## 可复用建议
- 显存 8GB：7B Q4_K_M + ctx 4096–8192，适合意图识别、参数提取、固定格式输出。
- 显存 16GB：14B Q4/Q5，工具调用更稳。
- 显存 24GB+：可考虑 32B Q4 或 MoE 模型，但首 token 延迟要实测。
- 把本地模型当“执行小任务”节点，不要让它做长链条规划；复杂规划可拆分成多个小模型调用，或保留云端模型。
- 写一个 startup script 固化 `OLLAMA_NUM_PARALLEL`、`OLLAMA_KEEP_ALIVE`、模型版本，避免系统更新后行为漂移。

## 总结
本地 LLM 部署的关键不是模型越大越好，而是让一个可预测的本地端点稳定服务于 Agent/MCP。控制好显存、上下文、量化等级和输出格式，家用电脑就能承担不少自动化任务；但工具调用和复杂推理仍要保守设计，必要时降级为 JSON 解析。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/f93d539143892a71.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/cc39c3149bcc2e33.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/bec287f2c4bd1a08.png)

