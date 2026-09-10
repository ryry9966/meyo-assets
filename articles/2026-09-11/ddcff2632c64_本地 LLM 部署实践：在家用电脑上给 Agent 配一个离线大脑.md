---
title: 本地 LLM 部署实践：在家用电脑上给 Agent 配一个离线大脑
feedId: 36956
source: 综合讨论
publishedAt: 2026-09-11
---

## 背景

做 Agent 开发有个现实痛点：调试循环烧 API。一次多轮工具调用轻松吃掉几万 token，反复跑测试用例时账单和限流都很磨人。对 OpenClaw 用户来说，本地部署 LLM 最实际的价值不是替代云端旗舰模型，而是提供一个**随时可用、不计费、数据不出门**的实验端点。

先校准预期：家用电脑跑本地模型，适合轻量任务、工具调用调试、隐私敏感数据；复杂长链推理仍建议回退云端。

## 问题

三个核心问题决定了成败：

1. **硬件决定上限**：显存直接限制模型规模，选错档位会陷入"要么跑不动、要么太慢"。
2. **量化选型**：Q4/Q5/Q8 在质量与显存之间权衡，新手容易被参数表劝退。
3. **接入质量**：Agent 框架依赖 function calling，而小模型的工具调用稳定性差异极大。

## 做法

**第一步：盘点硬件。** 8GB 显存 → 7~8B Q4；12~16GB → 14B Q4 或 8B Q6；24GB 以上可上 32B Q4。纯 CPU + 32GB 内存也能跑 7~8B Q4，速度约每秒几个 token，只适合低频调用。

**第二步：选运行时。** 推荐从 Ollama 起步，一行安装、内置 OpenAI 兼容接口（11434 端口）。想榨性能再换 llama.cpp 或 vLLM（后者需要 Linux + NVIDIA）。

**第三步：拉模型验证。**

```bash
ollama pull qwen2.5:7b-instruct-q4_K_M
ollama run qwen2.5:7b-instruct-q4_K_M
```

先聊几轮，确认速度和质量在可接受范围。

**第四步：接入 OpenClaw。** 在模型配置里把 `base_url` 指到 `http://localhost:11434/v1`，API key 随意填。MCP server 不用动——它们跑在本机，走的是工具协议，与模型解耦。

**第五步：实测工具调用。** 准备一组固定用例（读文件、发 HTTP 请求、查数据），先裸测模型，再挂进 Agent 全链路跑。

## 踩坑点

- **上下文静默截断**：Ollama 默认上下文偏小，Agent 的 system prompt + 工具定义很容易超。用 `num_ctx` 显式调大，但注意 KV cache 会额外吃显存，调太大直接 OOM。
- **Function calling 不要想当然**：不是所有模型都稳定支持。嵌套 JSON 参数的输出错误率在小模型上明显升高，一定先裸测再接入。
- **CPU/GPU 混合推理速度悬崖**：显存不够时部分层 offload 到 CPU，速度断崖式下跌，而 Agent 多轮循环会放大这个延迟。
- **工具注册过多**：小模型在工具数量多时容易幻觉出不存在的参数。砍到 5 个以内，把工具 description 写清楚。
- **Embedding 别忘了**：本地做 RAG 需要单独起一个 bge/m3e 类 embedding 服务，主模型不负责向量化。

## 可复用建议

1. **从 8B Q4 开始打通链路**，再考虑升规格，不要上来就追最大的模型。
2. **双模型路由**：本地小模型干摘要、分类、抽取这类轻活，复杂推理走云端，OpenClaw 的多 provider 配置天然支持按任务切换。
3. **维护个人评测集**：20 条工具调用用例，换模型前必跑一遍，几分钟避免几小时的排障。
4. **记录硬件基线**：把每个模型组合的 tokens/s 和显存占用记进表格，下次选型直接查。
5. **保留云端逃生通道**：配置里留一个一键切回云 provider 的开关，演示和交付场景不赌本地稳定性。

## 总结

本地 LLM 部署本质是三件事对齐：硬件上限、量化选型、Agent 接入。对 OpenClaw 用户而言，本地模型最大的意义是获得一个可自由折腾、可断网运行、隐私可控的实验环境。从一张 8B Q4 模型开始，把工具调用链路跑通，你对 Agent 能力边界的理解会比看十篇评测更具体。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-11/10012c686a9e4eb7.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-11/927fa036e980e804.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-11/873043f9fc91ebf0.png)

