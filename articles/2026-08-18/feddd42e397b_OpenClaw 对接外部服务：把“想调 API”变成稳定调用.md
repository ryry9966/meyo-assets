---
title: OpenClaw 对接外部服务：把“想调 API”变成稳定调用
feedId: 33767
source: 综合讨论
publishedAt: 2026-08-18
---

## 背景：Agent 最后一步总是要碰外部服务

在 OpenClaw 里做自动化，模型负责推理和决策，但真正干活的是外部服务：查订单、发通知、拉报表、写工单。常见的接入方式有三种：

1. 直接让模型生成 HTTP 请求；
2. 在 OpenClaw 插件里写本地工具函数；
3. 通过 MCP server 暴露外部服务能力。

第一种最灵活，也最不可控。模型可能拼错 URL、忽略鉴权、把错误响应当成事实继续推理。工程上更稳的做法，是把外部 API 封装成 OpenClaw 可发现的工具，让 Agent 只面对一个干净的、结构化的接口。

## 问题：外部 API 并不是天生适配 Agent

普通 HTTP API 通常返回给前端或脚本用，字段命名随意、错误语义不统一、分页靠约定、限流靠调用方自己处理。Agent 调用时，如果收到一个 500 或 HTML 错误页，很可能开始“脑补”原因，甚至把错误信息写进最终结果。

所以对接外部服务的核心不是“能不能调通”，而是：**握手失败时，Agent 能不能快速理解发生了什么，并且不再继续用错误信息往下走。**

## 做法：一层薄封装，而不是裸 HTTP

在 OpenClaw 里对接外部服务，我建议按下面步骤来。

### 1. 先定工具边界

一个工具只做一件事。比如“按订单号查询订单状态”是一个工具，“批量导出订单”是另一个工具。不要写一个万能 `call_api` 工具，让模型自己填 endpoint 和参数，那样等于把风险又推回给模型。

例如在 OpenClaw 的工具定义里，输入 schema 只暴露必要字段：

```json
{
  "name": "get_order_status",
  "description": "查询单个订单状态",
  "parameters": {
    "type": "object",
    "properties": {
      "order_id": { "type": "string" }
    },
    "required": ["order_id"]
  }
}
```

### 2. 封装一层 provider 函数

在插件或 MCP server 内部，用一个独立函数承载所有 HTTP 细节：base URL、鉴权、超时、重试、错误分类。

以 Python 为例：

```python
import os, httpx

def get_order_status(order_id: str) -> dict:
    base = os.environ["ORDER_API_BASE"]
    token = os.environ["ORDER_API_TOKEN"]
    try:
        resp = httpx.get(
            f"{base}/orders/{order_id}",
            headers={"Authorization": f"Bearer {token}"},
            timeout=10,
        )
        resp.raise_for_status()
        data = resp.json()
        return {"ok": True, "data": data}
    except httpx.TimeoutException:
        return {"ok": False, "error": "timeout", "hint": "订单服务响应超时，请稍后重试"}
    except httpx.HTTPStatusError as e:
        code = e.response.status_code
        hint = "订单不存在" if code == 404 else f"订单服务返回 {code}"
        return {"ok": False, "error": str(code), "hint": hint}
    except Exception as e:
        return {"ok": False, "error": "unknown", "hint": str(e)}
```

### 3. 固定返回结构

所有工具都返回同一个 envelope：

```json
{
  "ok": true,
  "data": {},
  "error": null,
  "hint": null
}
```

这样 Agent 不用猜返回结构。`ok=false` 时，模型可以直接把 `hint` 转成用户能听懂的话。

### 4. 在 OpenClaw 中注册

如果走 MCP，就把上面的函数暴露成 MCP tool；如果走插件，就按 OpenClaw 的插件机制注册。注意不要在工具描述里写太多实现细节，重点写清楚“什么时候该调用、返回什么”。

## 踩坑点

### 1. 错误响应被当成业务数据

这是最常见的问题。外部服务返回的 HTML、网关错误页、登录页，如果不加判断直接传给模型，模型可能把“请输入验证码”当成订单状态。解决方式是检查 `Content-Type` 和状态码，非 JSON 或非 2xx 一律走错误分支。

### 2. 超时设置太随意

Agent 通常有整体任务超时。外部 API 超时时间不要跟 Agent 超时时间一样长。一般建议外部请求 timeout 设在 5~15 秒，并在错误提示里带上“稍后可重试”。

### 3. 分页被忽略

很多列表接口默认只返回第一页。如果模型拿第一页 20 条说“这就是全部”，就会漏数据。封装时要么在工具里循环拉全量，要么明确返回 `has_more`，让模型决定是否继续调用。

### 4. 鉴权信息读不到

插件进程可能不继承你 shell 里的环境变量。OpenClaw 启动插件时，要显式把 `ORDER_API_TOKEN` 这类变量注入，不要依赖 `.env` 自动加载。

### 5. 让模型拼 query string

不要让模型拼 `?start_time=2024-01-01&status=paid`。模型容易把时间格式、URL 编码、布尔值写成错误形式。工具参数里接收结构化字段，由 provider 函数构造请求。

## 可复用建议

- 外部服务先包一层 provider，再注册为工具，不要直接暴露裸 HTTP。
- 每个外部 API 统一返回 `{ok, data, error, hint}`。
- 错误分类至少区分：参数错误、鉴权失败、超时、服务端错误、未知错误。
- 对外部请求加上 `request_id` 或 `trace_id`，方便联调。
- MCP 场景区分读写工具权限，避免 Agent 误调用写操作。
- 联调先用 curl 或脚本跑通，再让 Agent 调用，不要上来就让模型试。

## 总结

OpenClaw 对接外部服务，关键不是“把 API 接上”，而是把外部服务包裹成 Agent 能稳定理解的工具。薄封装、统一返回、错误分类、超时和分页处理，这些看起来普通的工程细节，决定了 Agent 是稳定执行，还是偶发抽风。握手要稳，边界要清，错误要能被读懂。

---

