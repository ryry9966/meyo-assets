---
title: Agent 与 API 的握手：OpenClaw 怎么对接外部服务
feedId: 36388
source: 综合讨论
publishedAt: 2026-09-07
---

## 背景

LLM 本身只会生成文本，Agent 的真正能力来自它能调用什么。在 OpenClaw 里，这层"握手"通常由 MCP 或插件机制承担：把一个 HTTP 接口包装成 Agent 可理解的 tool，由模型决定何时调用、传什么参数。听起来简单，但从"本地能跑通"到"稳定可用"，中间有不少工程细节值得展开。

## 问题

我们最早对接内部工单系统 API 时踩过一轮典型的坑：模型经常猜错参数名；接口返回的 JSON 动辄几十 KB，几轮对话就把上下文撑爆；网络抖动时 Agent 自动重试，结果创建了两条重复工单。这些都是对接层的问题，不是模型的问题——对接层做不好，模型越聪明，行为越不可控。

## 做法

以接入一个"创建工单"API 为例：

1. **定义清晰的 tool schema**。参数名、类型、必填项、枚举值尽量显式。description 要写给"新来的同事"看，而不是写给自己——模型对参数语义的理解完全依赖这段描述。
2. **在 MCP server 里包一层**。不要让 Agent 裸调 HTTP。封装层统一负责鉴权、超时、重试和错误归一化，把 401/403/429/5xx 翻译成模型能据此行动的短文本。
3. **凭证走环境变量或密钥管理服务**，配置文件里只留引用，不落明文。
4. **裁剪返回结果**。只把模型需要的字段（如 ticket_id、status）回传，原始 JSON 写日志，不进上下文。
5. **写操作配幂等键**。用会话 ID + 意图哈希生成 idempotency_key，重试就不会产生重复副作用。

## 踩坑点

- **description 太含糊**。`id: 用户 ID` 不如 `id: 工单系统内的数字编号，形如 2024xxxx，可先调用 list_tickets 获取`。改完后模型猜参数的次数明显下降。
- **原始响应直接塞给模型**。上下文成本高，且大段 JSON 会稀释真正有用的信号。
- **重试策略一刀切**。GET 可重试；POST 必须配幂等键；429 要读 Retry-After；5xx 用指数退避。
- **超时设太长**。等待期间 Agent 是阻塞的，30 秒超时对交互是灾难，建议 10 秒内失败并返回可读错误。
- **日志里打了 Authorization 头**。排查问题时把密钥带进了日志系统，这个我们真的发生过，务必在封装层做脱敏。

## 可复用建议

- 每个 API 封装成独立 MCP tool，别做"万能调用器"——工具越多越笼统，模型选择准确率越差。
- 错误信息写给模型看："缺少 project_id，可调用 list_projects 获取"远比 "400 Bad Request" 有用。
- 上线前用一组固定的测试 prompt 做回归，改 schema 后尤其要跑。
- 工具超过 20 个后按场景分组挂载，避免工具列表本身吃掉太多上下文。

## 总结

对接外部服务的本质不是"调通 HTTP"，而是给模型一个语义清晰、行为可控的接口面：schema 是契约，封装层是安全垫，幂等与可观测性是生产化的底线。这三件事做扎实，OpenClaw 的 Agent 才算真正和外部服务握上了手。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/b3a2fa3dd9bb9948.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/bc170e0a055df32e.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/5941733cb66f6faf.png)

