---
title: Agent 与 API 的握手：OpenClaw 对接外部服务的工程化实践
feedId: 34027
source: 综合讨论
publishedAt: 2026-08-21
---

## 背景

在 OpenClaw 上跑 Agent，只靠模型内置知识远远不够。查订单、建工单、发通知、写文档，都需要访问外部服务。但“能调通一次 API”和“能在 Agent 里稳定调用”是两件事。

这次记录的是把一个外部工单系统接入 OpenClaw 的过程。核心不是让模型学会发 HTTP 请求，而是把外部 API 包装成受控、可观测、可测试的工具。

## 问题

直接让模型自由调用外部 API，通常会遇到四类问题：

1. 模型自由拼 URL 和参数，结果不可控。
2. API Key 进入上下文或日志，容易泄露。
3. 外部 API 的返回体、错误码、限流不被处理，Agent 被垃圾信息淹没。
4. 超时、重试、分页逻辑缺失，上线后频繁失败。

## 做法

### 1. 选择接入路径

轻量 API 用 OpenClaw 自定义 tool 就够了；已有 MCP server 可以直接挂；复杂业务建议写一个本地桥接服务，OpenClaw 调用本地 tool，再由桥接服务转发外部 API。本次采用“自定义 tool + 本地桥接服务”。

### 2. 先用 curl 验证

写工具前，先把外部 API 的请求、响应、鉴权、错误码、限流头用 curl 跑一遍。这一步能避免很多后续返工。

### 3. 定义 tool schema

在 OpenClaw 的 tool 配置里，把参数约束写清楚。模型只负责决策，不负责猜接口形状。示例：

```yaml
tools:
  - name: create_ticket
    description: 在工单系统创建一个工单。仅在用户明确要求时使用。
    parameters:
      type: object
      properties:
        title: {type: string, description: 工单标题}
        priority: {type: string, enum: [low, normal, high]}
        assignee_id: {type: string, description: 可选，处理人ID}
      required: [title, priority]
```

### 4. 密钥只放环境变量

API Token 放在 `.env`，桥接服务从 `process.env` 读取。不要在 tool 描述、prompt 或 YAML 配置里写任何密钥。

### 5. 薄封装 HTTP client

统一加超时、重试、trace id。示例：

```ts
async function callApi(path, payload) {
  const res = await fetch(`${BASE}${path}`, {
    method: 'POST',
    headers: {
      Authorization: `Bearer ${process.env.API_TOKEN}`,
      'Content-Type': 'application/json',
      'X-Trace-Id': crypto.randomUUID()
    },
    body: JSON.stringify(payload),
    signal: AbortSignal.timeout(8000)
  });

  if (!res.ok) {
    const text = await res.text().catch(() => '');
    throw new Error(`api_error:${res.status}:${text.slice(0, 200)}`);
  }

  return res.json();
}
```

### 6. 响应裁剪与错误映射

外部 API 可能返回 60 个字段，但工具只需要返回 `id`、`status`、`title`、`created_at`、`assignee`。列表默认 `limit=20`，并处理 `next_cursor`。错误码映射成可读信息，不要把原始堆栈抛给模型。

### 7. OpenClaw 调用内部服务

让 OpenClaw 调用本地桥接服务，而不是直接请求外部 API。这样便于单元测试、mock 和切换环境。

## 踩坑点

- **API Key 写进 tool description**：日志一开直接泄露。密钥只进环境变量。
- **让模型自由构造时间参数**：比如“最近三天”被模型自己算成错误时间戳。应该由工具内置时间计算。
- **返回体不裁剪**：模型上下文被大 JSON 打爆，后续动作丢失。
- **401/403 反复重试**：账号可能被锁。重试只针对 429 和 5xx，且只重试一次并退避。
- **时区问题**：外部 API 返回 UTC，Agent 按本地日期过滤导致漏单。统一在桥接层显式标明时区。
- **分页只取第一页**：总数看起来永远只有 20。

## 可复用建议

- 一个外部服务一个 tool，宁可多几个，不要让一个 tool 包打天下。
- 先加 `--dry-run` 参数，写操作先空跑验证。
- 日志只记 trace id、方法、状态码、耗时，不记 Authorization 和请求体。
- 用 OpenAPI 导入可以快速生成 schema，但导入后必须人工裁字段、加约束。
- 创建类 API 尽量带 idempotency key，避免 Agent 重试产生重复数据。

## 总结

OpenClaw 对接外部服务，本质不是让 Agent 学会发 HTTP 请求，而是把 API 包装成可约束、可观测、可测试的工具。模型负责决策，工具层负责保证调用可靠。这个边界守住，后面接十个外部服务也不会乱。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/57321106f4f338e7.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/1bada11adec6ca7e.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/4dbac4adc66e98d8.png)

