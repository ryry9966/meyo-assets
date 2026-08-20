---
title: 本地 LLM 部署：在家用电脑上跑大模型的实践指南
feedId: 33928
source: 综合讨论
publishedAt: 2026-08-20
---

## 背景

OpenClaw 生态里，Agent、MCP、插件和自动化脚本默认大多走云端大模型 API。但很多实践场景需要数据不出本机、离线运行，或者单纯不想为每次工具调用付 token 费。家用电脑跑本地 LLM 已经可行，但真正难的不是“能跑”，而是跑起来之后能被 Agent 稳定调用——尤其是工具调用、结构化输出和长时间任务。

这篇文章不堆参数表，只记录我在一台普通家用机器上给 OpenClaw 接本地 LLM 的工程化过程。

## 问题

家用电脑的限制很明确：显存小、内存带宽有限、没有企业级推理栈。常见误区是直接下载一个 70B 模型尝试跑，结果要么 OOM，要么生成速度慢到无法做多轮 Agent 任务。另一个更隐蔽的问题是：很多模型能聊天，但工具调用不稳定，输出格式经常不符合 OpenClaw/MCP 的预期，导致 Agent 第一步就失败。

所以部署目标不是“跑大模型”，而是“跑一个能被自动化流程可靠调用的本地推理服务”。

## 做法与步骤

### 1. 硬件与模型匹配

先确认机器配置。个人经验：

- 8GB 显存：优先 7B–8B 参数模型，Q4_K_M 或 Q5_K_M 量化。
- 16GB 显存：可以尝试 13B–14B 模型，Q5_K_M 或 Q8_0。
- 32GB 内存 + 6GB 显存：用 llama.cpp 的 CPU/GPU 混合推理，跑 13B 也勉强能用，但速度要实测。

模型选择上，不要只看通用榜单，重点看工具调用能力。我目前更倾向 Qwen2.5-Instruct 系列、Mistral 系列以及社区里专门做过 function calling 微调的模型。量化版本至少 Q4_K_M，Q3 及以下在工具调用时很容易胡说。

### 2. 推理后端选型与启动

推荐 Ollama 或 llama.cpp server，暴露 OpenAI 兼容接口。Ollama 上手快，但自定义 chat template 和参数稍微绕；llama.cpp server 更底层，适合需要 grammar 约束和细粒度控制的用户。

以 Ollama 为例，拉取模型后先创建一个 Modelfile 固定参数：

```dockerfile
FROM qwen2.5:7b-instruct-q4_K_M
PARAMETER temperature 0.3
PARAMETER num_ctx 8192
PARAMETER stop "<|im_end|>"
PARAMETER stop "<|endoftext|>"
```

然后：

```bash
ollama create openclaw-local -f Modelfile
ollama run openclaw-local
```

服务默认监听 `http://127.0.0.1:11434`。

### 3. 接入 OpenClaw

在 OpenClaw 的模型配置里，不要用云端格式，改用 OpenAI 兼容端点：

```yaml
model_provider: openai_compatible
base_url: http://127.0.0.1:11434/v1
model: openclaw-local
api_key: "not-needed"
```

关键是接下来测试工具调用。给 Agent 配一个最简单的 MCP 工具或本地函数，让它执行“查询当前目录文件列表”之类的任务，观察它是否能按 schema 输出参数。

### 4. 工具调用与结构化输出

如果工具调用不稳定，先检查两件事：

- 模型是否真的支持 function calling；
- chat template 是否正确。

Ollama 有时候会使用默认模板，导致工具描述没有被正确拼进提示词。建议在 Modelfile 里显式写入模型的 TEMPLATE，或者换用 llama.cpp server 并开启 `--jinja` 使用原版模板。

对于需要严格 JSON 输出的场景，llama.cpp 可以直接加 grammar 文件限制输出结构。这在 OpenClaw 的插件或自动化节点里非常有用，比反复重试提示词可靠得多。

## 踩坑点

1. **显存/内存被 num_ctx 打爆**  
   上下文长度不是越大越好。7B 模型从 4096 涨到 32768，KV cache 可能直接把显存吃满。建议从 4096–8192 起步，确认稳定后再增加。

2. **并发一高就 OOM**  
   本地推理服务不像云端能弹性扩。家用显卡并发 1–2 次请求还行，超过就排队或崩溃。OpenClaw 里如果有并行 Agent，记得限制并发或串行执行。

3. **忽略 chat template**  
   模型输出工具调用时多一个换行、少一个引号，都可能让 Agent 解析失败。务必确认后端用的是模型原版模板，而不是默认 fallback。

4. **工具描述太多撑爆上下文**  
   MCP 服务器经常暴露几十个工具，全量塞进 prompt 会让小模型吃不下。宁可手动裁剪工具集，只暴露当前自动化流程需要的那几个。

5. **温度设太高**  
   聊天可以调高温度增加多样性，但工具调用温度建议 0.1–0.3，否则参数很容易生成错误。

## 可复用建议

- **固定版本与量化类型**：不要用 latest 标签，写死模型名和量化后缀，避免某天自动更新后行为变化。
- **记录一组 smoke test**：至少准备 3 个用例——普通问答、单工具调用、多工具组合。每次换模型或参数后重跑。
- **把本地推理服务当系统服务管理**：用 systemd 或 pm2 托管，设置重启策略和日志轮转。Agent 运行到一半推理服务挂掉是最难查的问题。
- **逐步扩大上下文**：先跑通多轮轻任务，再往上加 num_ctx，不要一上来就上长文档。
- **保留云端回退开关**：本地模型调试失败时，OpenClaw 能临时切回云端 API，避免自动化流程完全不可用。

## 总结

家用电脑部署本地 LLM 给 OpenClaw 用，可行性已经足够高，但工程重点不在“跑起来”，而在“稳定地被 Agent 调用”。选对量化模型、固定 chat template、控制上下文长度、限制工具集，这些细节比盲目追求更大参数更重要。

本地模型不是云端 API 的平价替代，而是一种可控、离线、数据私有的推理方式。当你把 OOM、JSON 解析失败、模板不匹配这些问题都踩过一遍之后，它才会真正成为自动化链路里可靠的一环。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-20/51c36c855b4ca935.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-20/74b744c0a6d91ce1.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-20/7f8fa58850367129.png)

