---
title: Agent 与 API 的握手：OpenClaw 对接外部服务的工程化拆解
feedId: 34310
source: 综合讨论
publishedAt: 2026-08-23
---

## 背景

Agent 调用外部服务时，最容易踩进的误区是：把 API 文档丢给模型，让它自己拼请求。实际跑起来常出现参数名写错、token 被当普通文本输出、超时卡住、分页翻不完、返回一大段 JSON 把上下文打爆。

在 OpenClaw 里对接外部服务，本质不是“给模型一个 URL”，而是给 Agent 一个边界清晰的工具。LLM 适合做决策和摘要，不适合当精确的 HTTP 客户端。

## 问题

外部 REST API 有几个天然不稳定因素：认证方式不统一、限流策略不同、错误语义不明确、响应体积不可控。直接让 Agent 裸调，等于把这些问题全部丢给模型，出问题后很难稳定复现。

## 做法/步骤

### 1. 先定义动作，而不是定义接口

不要问“怎么对接这个 API”，先问“Agent 需要完成什么动作”。例如“查询订单状态”“创建工单”“拉取最近 7 天日志”。每个动作对应一个工具，而不是把整个 API 暴露给模型。

### 2. 凭证只进环境变量

Access token、API key、签名密钥一律放环境变量或 OpenClaw 的 secret 配置里，不让模型看到，也不写进工具描述。工具描述里只写用途、参数和返回语义。

### 3. 写一个薄 handler

在 OpenClaw 里注册工具时，handler 负责所有“不智能”的部分：固定 base URL、拼路径、加认证头、设置超时、处理重试、裁剪响应。

一个简化示例：

```python
import os
import json
import httpx
from pydantic import BaseModel, Field

class GetOrderInput(BaseModel):
    order_id: str = Field(..., pattern=r"^[A-Z0-9-]+$")

async def get_order(args: GetOrderInput) -> str:
    async with httpx.AsyncClient(timeout=5.0) as client:
        r = await client.get(
            f"https://api.example.com/orders/{args.order_id}",
            headers={"Authorization": f"Bearer {os.environ['API_TOKEN']}"}
        )
    if r.status_code == 404:
        return "ORDER_NOT_FOUND"
    if r.status_code == 429:
        return "RATE_LIMITED"
    r.raise_for_status()
    data = r.json()
    return json.dumps({
        "id": data["id"],
        "status": data["status"],
        "amount": data["amount"],
    }, ensure_ascii=False)
```

核心不是代码本身，而是把“不确定性”关在 handler 里。

### 4. 注册为 OpenClaw 工具或 MCP server

如果是少量工具，直接注册到 OpenClaw 的工具列表即可；如果外部服务较多、需要在多个 Agent 之间复用，包一层 MCP server 更合适。无论哪种方式，模型看到的只是工具签名、参数 schema 和返回说明，而不是原始 HTTP 细节。

### 5. 错误语义化

把 HTTP 错误映射成模型能稳定理解的结果：

- `401` → `AUTH_FAILED`
- `429` → `RATE_LIMITED`
- `404` → `NOT_FOUND`
- `500` → `UPSTREAM_ERROR`

不要直接把 traceback 或完整响应抛给模型，否则它可能会模仿错误信息，或者把内部变量名带进下一步决策。

## 踩坑点

- **让模型直接拼 URL**：路径注入、版本乱填、参数名错误，排查起来很痛苦。
- **不设超时**：外部服务抖动时，Agent 会卡到用户以为它死了。
- **返回全量 JSON**：上下文窗口被迅速占满，模型开始丢关键字段，费用也上升。
- **盲目重试所有请求**：非幂等的 POST 请求被重试，可能产生重复订单或重复工单。
- **schema 不校验**：错误输入直接透传到 API，错误信息更难定位。
- **把 token 写进工具描述**：模型可能把凭证当成普通信息引用到输出里。

## 可复用建议

- **读写分离**：只读工具与写操作工具分开，写操作可以加确认或 human-in-the-loop。
- **裁剪返回字段**：优先返回状态枚举、ID、时间和错误码，而不是大段原始 JSON。
- **统一工具返回码**：例如 `TOOL_OK`、`TOOL_RATE_LIMITED`、`TOOL_AUTH_FAILED`，让模型决策更稳定。
- **记录 request_id**：每次请求都落日志并带 request_id，方便和服务方排障。
- **给分页设上限**：固定 page_size，设置 max_pages，避免 Agent 无限翻页。
- **版本化 schema**：外部 API 变更时，先更新工具定义，而不是靠提示词兜底。

## 总结

Agent 与外部服务的稳定对接，关键是提前收口：输入有 schema，输出有裁剪，错误有语义，凭证有隔离。在 OpenClaw 里，我更倾向于把外部服务封装成少量明确的工具或 MCP server，而不是让 Agent 充当 HTTP 客户端。这样 LLM 才能做它擅长的事，而 API 的握手保持确定和可复现。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/67d8503f3bd266f2.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/e5f33bd394cfa0f5.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/8f07cd0966296360.png)

