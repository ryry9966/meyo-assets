---
title: Agent 与 API 的握手：OpenClaw 对接外部服务的工程实践
feedId: 35454
source: 综合讨论
publishedAt: 2026-08-31
---

## 背景

OpenClaw 作为 Agent 运行时，真正能落地的场景很少只靠内置知识。查订单、读日历、发通知、操作内部系统，都依赖外部 API。把外部服务接进来，难点不在“能不能发 HTTP 请求”，而在怎么让模型稳定传参、怎么让失败不污染推理、怎么让密钥不泄漏。

如果只是给 Agent 一个通用 HTTP 工具，让它自己拼 URL、传 JSON，很快就会出现几类问题：模型把 500 当成“查询无结果”；一个慢接口把整个任务拖到超时；非幂等写操作被自动重试两次；API key 被写进提示词或日志。

所以这件事的本质，是在“不可靠的模型调用”和“有契约的外部服务”之间做一层可靠握手。

## 问题拆解

对接外部服务时，通常会撞上四类问题：

1. **参数不确定**：模型生成的字段可能多、少、类型错，外部 API 又要求严格契约。
2. **错误被误读**：超时、限流、网关错误如果原样丢给模型，它可能编造业务结果。
3. **认证泄漏**：token 直接放进工具描述、请求头或上下文，容易被打进日志。
4. **链路脆弱**：外部服务慢、挂、返回大 JSON，都会拖垮 Agent 循环。

## 做法/步骤

建议把每个外部服务封装成 OpenClaw 的工具或 MCP server，而不是暴露“万能 HTTP 工具”。

### 1. 定义工具契约

先写清楚工具名、描述和输入 schema。描述要说明“什么时候用、不做什么”，schema 用来约束模型参数。

```json
{
  "name": "query_order",
  "description": "按订单号查询订单状态。只用于用户明确提供订单号时。",
  "input_schema": {
    "type": "object",
    "properties": {
      "order_id": { "type": "string", "description": "订单号" }
    },
    "required": ["order_id"]
  }
}
```

### 2. 写适配层

在工具 handler 或 MCP server 内部完成参数校验、认证注入、HTTP 调用、错误捕获。不要让模型直接接触原始 API 细节。

### 3. 统一返回结构

给模型返回可读的结构化结果，而不是原始大 JSON：

```json
{
  "ok": true,
  "data": { "status": "shipped" },
  "error_code": null,
  "user_message": "订单已发货"
}
```

失败时用 `user_message` 告诉模型发生了什么，避免它自己猜。

### 4. 错误归一

把异常映射成稳定分类：

- `UPSTREAM_TIMEOUT`：可重试，但要带重试次数
- `BAD_REQUEST`：不可重试，参数有问题
- `RATE_LIMITED`：稍后重试
- `UNKNOWN`：未知错误，不要乱重试

### 5. 注册到 OpenClaw

通过 MCP server 或插件配置暴露工具。本地开发时先跑一个 mock server，确认模型能正确选工具、传参数、读返回。

### 6. 限流与观测

加耗时、重试次数、上游状态码日志，但日志要脱敏。外部依赖应有独立超时和熔断，不能无限等待。

## 踩坑点

- **超时**：不要用默认的长时间超时。Agent 循环里一个 30 秒的卡顿会让用户以为任务死了。建议 connect timeout 3-5 秒，read timeout 按业务设 10-20 秒，并支持 abort。
- **非幂等写操作**：创建订单、发送通知这类请求不要盲目自动重试。只有 GET 或明确幂等的 POST 才适合重试。
- **大返回体**：列表接口容易返回几 MB JSON。工具层要做截断、分页或摘要，只把关键字段给模型。
- **密钥位置**：API key 放服务端环境变量，不要写进工具描述、不要传给模型、不要在日志里打印请求头。
- **MCP server 生命周期**：如果 MCP server 进程挂了，工具会从 OpenClaw 消失。要加健康检查，必要时自动重启。
- **回调接口**：外部服务的 webhook 不适合 Agent 长连接场景。优先用 polling 或队列把异步结果拉回来。

## 可复用建议

- **优先 MCP**：标准工具 schema 可移植，换 Agent 运行时不用重写适配层。
- **客户端独立成模块**：外部 API 客户端脱离 OpenClaw 也能单元测试，不要和工具 handler 耦合。
- **mock 先行**：用录制或手写 fixture 做离线 mock，避免开发时反复打真实接口。
- **错误分类统一**：所有工具返回同一套 `error_code` 枚举，方便上游策略处理。
- **提示词约束**：在系统提示词里明确要求“工具失败时返回失败，不要编造结果”。

## 总结

OpenClaw 对接外部服务，不是拼 URL 和 token。稳定的做法是把每个外部 API 封装成有契约、有错误归一、有超时和重试策略的工具。模型负责意图和参数，工具层负责与外部世界握手。握得稳，Agent 才敢真正放进生产链路。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/688499965f810156.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/ebf33541a5d59b3a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/617cd9b44392577f.png)

