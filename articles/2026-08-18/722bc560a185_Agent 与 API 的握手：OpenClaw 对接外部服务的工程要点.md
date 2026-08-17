---
title: Agent 与 API 的握手：OpenClaw 对接外部服务的工程要点
feedId: 33671
source: 综合讨论
publishedAt: 2026-08-18
---

## 背景

Agent 如果只停留在“生成文本”，很难进入真实自动化场景。查订单、发告警、建工单、拉日志，这些动作最终都要落到外部服务上。OpenClaw 作为执行环境，真正有价值的部分不是模型本身，而是它能不能把一次决策稳定地转换成一次外部 API 调用。

但外部 API 通常不是为 Agent 设计的。它们有自己的鉴权方式、超时特性、错误码、非结构化返回，甚至文档和实际行为不一致。直接让 Agent 自由发 HTTP 请求，很容易出现越权调用、重复写操作、被一个慢接口拖死整个任务的情况。

所以问题不是“能不能调通”，而是怎么把不可控的 HTTP 调用，封装成 Agent 可以安全、稳定使用的工具。

## 做法与步骤

### 1. 先把外部服务封装成工具

不要让模型直接拼接 URL 和请求体。常见做法是在 OpenClaw 中注册工具，把外部服务的能力收敛成有限的几个动作，比如 `query_order`、`create_ticket`、`search_logs`。

一个最小示例，以自定义 handler 为例：

```python
from openclaw import tool

@tool(
    name="query_order",
    description="按订单号查询订单状态，只读操作，不会修改任何数据。",
)
def query_order(order_id: str) -> dict:
    resp = http_client.get(
        f"https://api.example.com/orders/{order_id}",
        timeout=5,
    )
    resp.raise_for_status()
    return normalize_order(resp.json())
```

工具描述要写清楚：它做什么、有什么副作用、参数含义。Agent 需要这些信息来决定是否调用，而不是靠猜。

### 2. 鉴权与密钥管理

API Token 不要写死在工具代码或配置里，更不要放进工具描述。建议统一从环境变量注入：

```yaml
headers:
  Authorization: "Bearer ${env:ORDER_API_TOKEN}"
```

如果 OpenClaw 部署在多个环境，短期 token 可以放 secret manager，不要在所有配置副本里散落长期密钥。写操作类的接口，尽量再加一层限制，比如只允许特定来源 IP、使用短期凭证或申请单独的低权限账号。

### 3. 超时、重试与幂等

外部 API 抖动是常态。每个工具都要设置独立超时，并且要小于 Agent 的调度超时，否则一个慢接口会卡住整个执行链路。

重试要分情况：

- 只读接口，比如查询、搜索，可以重试 1~2 次，采用短退避；
- 写操作，比如创建工单、扣库存、发消息，不要盲目自动重试。如果 API 支持幂等键，传入 `Idempotency-Key`；不支持的话，失败后让 Agent 确认状态后再决定是否重试。

### 4. 响应裁剪与 schema 归一

第三方 API 返回经常非常臃肿，甚至关键字段时有时无。不要把原始 JSON 直接丢回给 Agent，否则容易上下文爆炸，也会让决策不稳定。

建议每个工具返回一个稳定的结构，只保留 Agent 需要的字段：

```python
def normalize_order(raw: dict) -> dict:
    return {
        "order_id": raw.get("id"),
        "status": raw.get("state") or raw.get("status") or "UNKNOWN",
        "updated_at": raw.get("updatedAt"),
        "total": raw.get("amount", {}).get("value"),
    }
```

这样即使上游字段变更，也只影响 normalizer，不会直接影响 Agent 行为。

### 5. 错误映射

不要直接把 HTTP 异常堆栈抛给 Agent。把错误码映射成可执行的提示：

- `401/403`：凭证失效或权限不足，检查 token；
- `429`：被限流，稍后重试；
- `5xx`：上游服务不可用，标记为暂时失败；
- 业务错误码：按具体含义返回，比如“订单不存在”。

Agent 拿到可读的错误信息后，才可能做出正确恢复动作。

## 踩坑点

1. **写操作自动重试导致重复动作**。发券、扣款、群发消息这类接口，重试一次就可能造成真实损失。默认不要对非幂等 POST 做自动重试。
2. **每个工具各自创建 HTTP client**。连接池不共享，高并发下容易耗尽本地端口，也影响 TLS 复用。建议全局维护一个 client，通过工具调用传入。
3. **工具超时与 Agent 超时冲突**。常见现象是 Agent 侧先超时，但 API 实际还在执行，导致后续状态判断错乱。
4. **把 token 混进日志或 prompt**。工具请求日志要脱敏，尤其是 header 部分。
5. **信任 API 文档中的非空字段**。实际返回里 `null`、空字符串、缺失字段很常见，必须在 normalizer 里兜底。

## 可复用建议

- **先只读后写**。优先接入查询类 API，跑通稳定性后再接写操作。
- **用 mock server 做契约测试**。把外部 API 的典型响应、异常响应固定下来，验证工具封装和 Agent 决策。
- **统一 HTTP 封装**。超时、重试、错误映射、日志都放同一层，不要在业务工具里重复实现。
- **监控工具调用**。记录每个工具的状态码、延迟、失败原因，便于区分是上游问题还是自身配置问题。
- **工具描述写清副作用**。只读、写操作、幂等性、可能耗时，这些信息对 Agent 决策很重要。

## 总结

OpenClaw 对接外部服务，本质上不是“接个 URL 就能用”，而是把不可控的 HTTP 请求变成一个有边界的工具：鉴权清晰、超时可控、重试有策略、返回结构稳定、错误可理解。这样做之后，Agent 才可能可靠地使用外部能力，而不是每次都在真实 API 的混乱细节里翻车。

---

