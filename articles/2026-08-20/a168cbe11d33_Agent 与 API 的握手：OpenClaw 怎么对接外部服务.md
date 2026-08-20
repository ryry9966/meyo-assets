---
title: Agent 与 API 的握手：OpenClaw 怎么对接外部服务
feedId: 33899
source: 综合讨论
publishedAt: 2026-08-20
---

## 背景：为什么 Agent 需要一次“握手”

Agent 的推理能力再强，也不能凭空拿到订单状态、用户信息或第三方数据。外部 API 是 Agent 的触角，但直接把 API 文档丢给模型、让它自己拼 HTTP 请求，通常不可靠——参数乱填、鉴权方式搞错、错误处理失控是常态。

在 OpenClaw 里，外部服务主要有三种接入方式：**内置工具**、**自定义插件/工具**、**MCP Server**。其中，自定义工具适合轻量对接单个 API，MCP 适合把一组外部能力打包复用。本文以最基础的自定义 HTTP 工具为例，讲清楚一次可落地的对接过程。

## 问题：不是“能不能调通”，而是“握手是否稳定”

调通一个 API 很简单，curl 一下就行。但在 Agent 场景里，真正要解决的是：

- 模型如何知道该传什么参数？
- 外部 API 超时、返回异常时，Agent 如何优雅降级？
- 敏感信息（如 API Key）如何不进入模型上下文？
- 返回数据过大时，如何避免 token 爆炸？

这些问题不解决，API 对接只是“能跑”，不是“可用”。

## 做法：四步完成一次工程化对接

### 1. 定义工具契约

不要手写 prompt 描述 API，而是用结构化 schema 约束参数。OpenClaw 中自定义工具一般会包含 `name`、`description`、`parameters` 三部分。

```yaml
name: get_order
description: 查询订单详情，返回订单状态和金额。仅在用户提供订单号时使用。
parameters:
  type: object
  properties:
    order_id:
      type: string
      description: 订单号，形如 ORD-20240101-1234
  required: [order_id]
```

要点：`description` 要写清楚“什么时候用”，而不是泛泛说“查询订单”。参数的枚举、格式、示例尽量写全，能显著减少模型乱填。

### 2. 实现 API 调用 handler

把 HTTP 调用封装成独立函数，不要直接写在工具入口里。这样便于测试、替换和统一加日志。

```python
import httpx
import os

async def fetch_order(order_id: str):
    async with httpx.AsyncClient(timeout=5.0) as client:
        resp = await client.get(
            f"{os.getenv('ORDER_API_BASE')}/orders/{order_id}",
            headers={"Authorization": f"Bearer {os.getenv('ORDER_API_KEY')}"},
        )
    resp.raise_for_status()
    data = resp.json()
    # 只返回 Agent 真正需要的字段，避免把原始响应整个塞进上下文
    return {
        "order_id": data["id"],
        "status": data["status"],
        "amount": data["amount"],
        "currency": data.get("currency", "CNY"),
    }
```

注意：API 地址和密钥从环境变量读取，不要硬编码，也不要写进工具描述。

### 3. 注册到 OpenClaw

将上面两部分组装成 OpenClaw 能识别的工具配置。不同版本注册方式略有差异，但核心都是“schema + handler”的组合。MCP 场景则是在 OpenClaw 的 MCP 客户端配置里指向一个已有的 MCP Server，工具发现和调用由协议完成。

### 4. 联调与验证

至少要覆盖三类用例：

- 正常路径：传入合法订单号，返回精简 JSON。
- 异常路径：API 返回 404/500/429，确认 Agent 不会把堆栈信息吐给用户。
- 边界路径：参数缺失、格式错误，确认 schema 校验能拦下来。

## 踩坑点

### 1. 超时与重试被忽略

外部 API 抖动是常态。默认不设超时或无限重试，都会让 Agent 卡死。建议设置 3–5 秒超时，重试只针对幂等 GET，写操作不要盲目重试。

### 2. 鉴权信息泄露进上下文

工具描述、返回数据、错误信息里都不要包含 API Key 或 token。如果 API 返回里带有敏感头部或调试信息，要在 handler 里剥掉。

### 3. 返回数据过大

很多外部 API 返回几十个字段，模型根本不需要。返回前做裁剪，只保留对任务有用的字段。如果可能涉及分页，要在工具参数里显式支持 `page` 或 `cursor`，避免模型只拿第一页就下结论。

### 4. 错误信息不可操作

`raise_for_status()` 之后直接抛 500 给模型，模型只会复述“出错了”。把错误转成结构化信息：

```python
if resp.status_code == 404:
    return {"error": "order_not_found", "message": "订单不存在，请检查订单号"}
```

这样 Agent 能据此引导用户，而不是死循环重试。

### 5. 限流与退避

外部 API 有 QPS 限制时，工具层要做简单的退避或缓存。尤其是高频查询类接口，可以在 handler 里加一个短 TTL 缓存，减少重复调用。

## 可复用建议

- **API client 与工具层分离**：HTTP 调用、鉴权、重试逻辑放在独立模块，工具层只做参数校验和返回裁剪。
- **优先考虑 MCP**：如果外部服务已有 MCP Server，直接接入 OpenClaw 的 MCP 客户端，比重复写自定义工具更省心，也便于跨项目复用。
- **返回结构越简单越好**：Agent 需要的是“小而准”的信息，不是完整的 API 响应。裁剪字段、压缩嵌套、统一错误格式。
- **每个工具写一个最小测试**：不要求完整单测，但至少要有一个能调通真实 API（或 mock）的脚本，升级时能快速回归。
- **日志带 request_id**：在 handler 里记录每次调用的 request_id、状态码和耗时，排障时能快速定位是模型乱填参数还是外部 API 挂了。

## 总结

Agent 对接外部 API，本质是一次“契约握手”：工具 schema 是对模型的契约，handler 是对外部服务的契约，返回结构是对上下文的契约。把这三层边界划清楚，大部分稳定性问题都能提前避免。OpenClaw 的插件和 MCP 生态提供了很好的扩展基础，但真正决定对接质量的，还是工程细节——超时、裁剪、错误处理、密钥管理。把这些做扎实，Agent 才能从“能调 API”变成“能可靠地调 API”。

---

