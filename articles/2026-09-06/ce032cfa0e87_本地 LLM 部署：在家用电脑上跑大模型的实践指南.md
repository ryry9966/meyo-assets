---
title: 本地 LLM 部署：在家用电脑上跑大模型的实践指南
feedId: 36322
source: 综合讨论
publishedAt: 2026-09-06
---

## 背景

玩 OpenClaw 这类 Agent 框架时间长了，数据敏感性问题迟早会撞上：聊天记录、MCP 工具读到的本地文件、自动化任务里的日记和待办，全都经第三方 API 过一遍，总归不踏实。另外 Agent 循环调用量大，API 账单也涨得快。所以我花了两周把主力模型迁到家里一台 4060 Ti 16G + 64G 内存的机器上，把过程和教训记下来。

## 问题

本地部署不是"能跑起来"就完事。对 Agent 场景，真正的门槛是三个：

1. **工具调用质量**：小模型的 function call 经常输出坏 JSON 或漏参数，MCP 工具链一断，整个 Agent 就废了；
2. **上下文与显存的矛盾**：Agent 多轮循环会堆历史，32k 上下文的 KV cache 本身就吃掉好几个 G；
3. **交互延迟**：单用户场景下，首 token 延迟比吞吐量重要，一个 2 tok/s 的大模型在 Agent 循环里基本不可用。

## 做法与步骤

1. **选运行时**。图省事用 Ollama，OpenAI 兼容接口开箱即用；要精细控制用 llama.cpp 的 llama-server（记得开 `--jinja`，否则 chat template 不对，工具调用直接坏）；Linux + 好显卡可以上 vLLM。
2. **选模型档位**。8–12G 显存：7B–14B 的 Q4_K_M 够日常；16G 显存：14B 的 Q5/Q6，或 MoE 结构模型（如 30B-A3B 的 Q4）配合少量层卸到内存，激活参数少，速度依然可观；无独显的大内存 Mac 跑 MoE 也行。
3. **接进 OpenClaw**。把 provider 指到本地 endpoint（`http://127.0.0.1:11434/v1` 这类），先跑通纯对话，再逐个挂 MCP 工具验证。
4. **用真实任务压测**。别只看 benchmark。写个冒烟脚本：连续 10 轮工具调用 + 塞 8k token 上下文 + 要求严格 JSON 输出，通过率低于八成就换模型。

## 踩坑点

- Q4 量化对聊天没影响，但 Agent 的结构化输出会明显劣化，宁可上 Q5/Q6，也别为省那点显存牺牲质量；
- 上下文别无脑拉满，先算 KV cache 占用，给工具结果注入留 2G 显存余量；
- llama.cpp 的 `-ngl` 部分层卸载，在 PCIe 带宽差的机器上可能比纯 CPU 还慢，以实测为准；
- 长驻服务配 systemd/launchd 自动重启；Ollama 空闲默认卸载模型，Agent 第一轮调用会卡十几秒，把 `keep_alive` 设长一点；
- 模型文件别追新，版本一变 template 和行为就变，整套 prompt 要重调。锁版本，手动升级。

## 可复用建议

最务实的架构是**分层路由**：本地模型承担隐私敏感任务和高频低难度任务（日记整理、文件归档、简单问答），复杂推理仍走云端——OpenClaw 支持按任务配不同 provider，没必要一刀切全本地。另外把那个冒烟测试脚本留下来，每次换模型、升级运行时后跑一遍，比任何直觉都可靠。

## 总结

家用电脑跑本地 LLM 在 7B–32B 区间已经完全可行，但对 Agent 用户，评判标准只有一条：**工具调用是否稳定**。把部署当成持续运维而不是一次性折腾——锁版本、留测试、分层路由——这套组合拳下来，本地模型能承担日常大约六七成的调用量，隐私和账单两头都舒坦。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/4cb459b34c47c86b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/49bf644209aa5ae9.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/9a5f9c9656d6be9e.png)

