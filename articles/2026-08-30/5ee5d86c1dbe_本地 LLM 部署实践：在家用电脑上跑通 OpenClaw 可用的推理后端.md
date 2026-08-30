---
title: 本地 LLM 部署实践：在家用电脑上跑通 OpenClaw 可用的推理后端
feedId: 35418
source: 综合讨论
publishedAt: 2026-08-30
---

## 背景

对 OpenClaw、Agent、MCP 和自动化脚本用户来说，本地 LLM 的意义不是“替代闭源大模型”，而是提供一个数据不出本机、无 API 费用、可离线运行、延迟可控的推理后端。很多自动化任务对隐私敏感，或者调用频率高但任务相对简单，例如信息抽取、分类、格式化输出、工具调用指令生成。把这些任务交给本地模型，能让整套系统更稳定、成本更低。

## 问题

家用电脑硬件差异很大。常见机器是 NVIDIA 消费级显卡 8GB 显存，或 Apple Silicon 16GB 统一内存。直接跑 13B/14B 未量化模型基本不可能，必须做量化、控制上下文长度，并处理工具调用不稳定的问题。另一个常被忽略的点：OpenClaw 这类 agent 框架通常通过 OpenAI 兼容 API 接入模型，如果本地推理服务没有正确暴露接口，或者模型不支持 function calling，插件和自动化流程就会频繁失败。

## 做法/步骤

### 1. 硬件和模型选型

- NVIDIA 8GB 显存：优先跑 7B 参数模型的 Q4_K_M 或 Q5_K_M 量化版。
- NVIDIA 16GB 显存：可以跑 13B/14B 的 Q4_K_M，或 7B 的 Q8_0。
- Apple Silicon 16GB 统一内存：可以跑 7B/14B 的 MLX 或 GGUF 量化版，速度通常优于同价位 x86 笔电。

模型优先选指令微调且支持 function calling 的版本，例如 Qwen2.5-7B/14B-Instruct、Llama-3.1-8B-Instruct、Mistral-7B-Instruct-v0.3。不要选基础模型，它们不会按工具调用协议返回结果。

### 2. 推理引擎选择

对家用场景，Ollama 是最省事的起点。它一条命令拉模型、起服务，并默认暴露 OpenAI 兼容接口。llama.cpp server 适合想要细粒度控制参数的场景；LM Studio 适合图形化调试；vLLM 需要较多显存，家用单卡并发优势不明显。OpenClaw 只需要一个 OpenAI 兼容 base_url，因此 Ollama 足够。

### 3. 部署步骤

安装 Ollama 后，先设置环境变量：

```bash
OLLAMA_HOST=0.0.0.0:11434
OLLAMA_NUM_PARALLEL=1
```

家用显卡建议并行数设为 1，避免多个请求争抢 KV cache 导致 OOM。

拉取模型：

```bash
ollama pull qwen2.5:7b-instruct-q4_K_M
```

创建 Modelfile 调整上下文和生成参数：

```dockerfile
FROM qwen2.5:7b-instruct-q4_K_M
PARAMETER num_ctx 8192
PARAMETER temperature 0.1
PARAMETER top_p 0.9
```

重新创建并启动：

```bash
ollama create local-agent -f Modelfile
ollama run local-agent
```

测试接口：

```bash
curl http://localhost:11434/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"local-agent","messages":[{"role":"user","content":"hello"}]}'
```

在 OpenClaw 中添加自定义模型：base_url 填 `http://localhost:11434/v1`，模型名填 `local-agent`，关闭或按需开启流式输出。

### 4. 与 OpenClaw / Agent / MCP 结合

不要一上来就让本地模型直接调用复杂插件。先单独测试工具调用能力：给一个简单 JSON 输出任务，跑 20 次，看格式是否稳定。如果失败率高，优先用 MCP 服务器把工具执行与模型推理分离——OpenClaw 通过 MCP 调用工具，本地 LLM 只负责决策和参数生成。这样即使模型偶尔输出不规范，也不会直接卡死插件流程。

## 踩坑点

- **默认上下文太短**：Ollama 默认 `num_ctx` 是 2048，agent 多轮对话很容易截断。必须显式调到 8192 或 16384，同时监控显存占用。
- **量化模型工具调用不稳定**：7B Q4 模型输出 JSON 时经常多出注释、引号转义错误或字段缺失。解决办法：温度设到 0 或 0.1，改用 Q5_K_M 或 Q8_0，必要时加 grammar 约束。
- **Windows 下 CUDA 回退 CPU**：驱动版本和 llama.cpp 编译不匹配会导致推理速度慢几十倍。安装后先跑一个小 prompt，确认 GPU 利用率不为 0。
- **并发请求导致 OOM**：OpenClaw 后台可能同时发出多个请求，家用显卡 KV cache 不够，必须限制并行数。
- **不要盲目上大模型**：14B 在家用电脑上 token 速度可能只有每秒 3-5 个，agent 交互体验会很差。先确认 7B 是否满足任务，再决定是否升级。

## 可复用建议

- 固定量化版本和 Prompt 模板，避免升级模型后行为漂移。
- 用 litellm 或自建 API 网关统一管理本地模型，OpenClaw 只对接一个端点，以后换引擎不影响 agent 配置。
- 写一个压测脚本，用固定 prompt 测 100 次工具调用成功率，低于 90% 就换模型或加约束。
- 把本地 LLM 定位为“能力有限的可靠组件”：适合隐私敏感、离线、简单工具调用和结构化提取，不适合复杂多步推理或高难度代码生成。

## 总结

本地 LLM 部署不是“装上就能用”，而是需要同时处理选型、量化、上下文长度、工具调用稳定性四件事。对 OpenClaw 用户来说，Ollama + 7B/14B 指令模型的组合是比较务实的起点。配合 MCP 分离工具执行，可以显著降低模型输出不稳定对系统的影响，把家用电脑真正变成一个可用的 agent 推理节点。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/1a6b1420c07c27d5.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/f68e9e2cada49eb3.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/4e1a5e5b4c61f0e8.png)

