---
title: Agent 与 API 的握手：OpenClaw 对接外部服务的工程化实践
feedId: 33151
source: 综合讨论
publishedAt: 2026-08-14
---

## 背景：为什么“直接让 Agent 调 HTTP”容易翻车

OpenClaw 的 Agent 能力很强，但很多实践者在接入真实业务时会发现：模型很会“说话”，一旦让它直接构造 HTTP 请求去调外部服务，就开始不稳定。要么参数乱拼，要么遇到 500 就疯狂重试，要么把密钥写进工具描述里。

常见的外部服务包括：内部 CRM、工单系统、支付网关、数据库 API、企业微信/钉钉通知、第三方数据接口等。这些服务通常有鉴权、超时、限流、幂等要求，而 LLM 在原始 HTTP 层直接操作时，很难稳定遵守这些约束。

所以在 OpenClaw 里对接外部服务，核心不是“让模型会发请求”，而是**把外部服务封装成稳定、可校验、带清晰副作用的工具契约**，再通过 MCP 或插件暴露给 Agent。

## 问题拆解

实际对接中，主要有四类问题：

1. **鉴权与密钥管理**：密钥容易泄漏在 prompt、日志或工具描述里。
2. **Schema 不稳定**：模型自由构造 URL 和参数，容易出现注入、拼错 path、字段类型错误。
3. **超时与重试**：外部 API 卡住会阻塞 Agent 执行循环；无脑重试会导致写操作重复提交。
4. **错误处理不可控**：外部服务返回的非结构化错误直接透传给模型，模型容易编造结果或陷入重试循环。

解决思路是把这些复杂度下沉到适配层，让模型只看到业务动作，而不是原始 HTTP。

## 做法：适配层 + MCP/插件暴露

以接入一个 CRM 工单服务为例。

### 第一步：定义工具边界

先确认 Agent 需要哪些业务动作，例如：

- 创建工单：`create_ticket(title, priority, content)`
- 查询工单状态：`get_ticket_status(ticket_id)`
- 添加备注：`add_ticket_comment(ticket_id, comment)`

不要暴露“调用任意 CRM API”这种宽泛能力。

### 第二步：写一个薄适配层

在 OpenClaw 的工具目录或插件里，写一个 service 模块，集中处理鉴权、超时、重试和错误归一化。

```python
class CRMClient:
    def __init__(self, base_url, api_key):
        self.base_url = base_url
        self.headers = {"Authorization": f"Bearer {api_key}"}

    def create_ticket(self, title, priority, content):
        payload = {
            "title": title,
            "priority": priority,
            "content": content,
            "idempotency_key": generate_idempotency_key(),
        }
        try:
            resp = requests.post(
                f"{self.base_url}/tickets",
                json=payload,
                headers=self.headers,
                timeout=(3, 10),  # connect timeout 3s, read timeout 10s
            )
            resp.raise_for_status()
            return resp.json()
        except requests.Timeout:
            return {"ok": False, "code": "TIMEOUT", "message": "CRM 服务超时", "retryable": True}
        except requests.HTTPError as e:
            return normalize_http_error(e)
```

关键点：**统一返回结构** `{ok, code, message, retryable}`，不要直接抛异常给 Model。

### 第三步：注册为 MCP 工具

把适配层方法包装成 MCP tool，给出清晰的 JSON Schema 和描述。

```json
{
  "name": "create_ticket",
  "description": "在 CRM 中创建工单。该操作有副作用，会真实创建工单。返回 ok=false 且 retryable=true 时可重试，否则不要重复调用。",
  "inputSchema": {
    "type": "object",
    "properties": {
      "title": {"type": "string", "description": "工单标题"},
      "priority": {"type": "string", "enum": ["low", "medium", "high"]},
      "content": {"type": "string", "description": "工单内容"}
    },
    "required": ["title", "content"]
  }
}
```

在 OpenClaw 配置中加载该 MCP server 或插件，重启后 Agent 就能调用。

### 第四步：先 CLI 验证，再交给 Agent

不要一上来就让 Agent 调。先在命令行或测试脚本里手动调用工具，确认鉴权、超时、错误返回都符合预期。然后再让 Agent 执行真实任务。

## 踩坑点

1. **密钥写进工具描述或 prompt**  
   模型会把工具描述里的内容当成可读文本，很容易在回答中“复述”密钥。密钥应放在环境变量或 secret store 中，工具描述里只写“使用已配置的鉴权信息”。

2. **Schema 太宽泛**  
   如果让模型自由传 URL path、headers 或任意 JSON body，等于把 HTTP 客户端暴露给模型。外部服务可能被错误调用，甚至产生安全风险。应把 path、method、headers 固化在适配层，模型只能选择预设动作和参数。

3. **超时设置不合理**  
   默认 requests 可能无限等待，导致 Agent 卡死。建议 connect timeout 3-5s，read timeout 10-15s，并区分对待。如果外部服务本身较慢，可以异步处理，但不要让同步工具长时间阻塞。

4. **错误信息直接透传**  
   外部 API 返回的 HTML 错误页、堆栈信息或长 JSON 直接给模型，会干扰判断。适配层应归一化为简洁的 `{ok, code, message, retryable}`，并限制 message 长度。

5. **写操作没做幂等**  
   创建工单、发通知、扣款等操作，如果超时后 Agent 重试，可能产生重复数据。应在适配层加入幂等键或去重逻辑，并在工具描述里明确“该操作可安全重试”或“不可重复调用”。

6. **MCP 连接失败导致整个 session 不可用**  
   MCP server 挂了会让所有工具调用失败。建议对关键外部服务提供降级策略：例如返回 `{ok: false, code: "SERVICE_UNAVAILABLE", retryable: false}`，让 Agent 明确知道当前不可用，而不是一直重试。

## 可复用建议

- **薄适配层原则**：外部 API 的复杂度只出现在适配层，模型看到的永远是业务动作。
- **工具描述写清副作用**：只读、有副作用、幂等性、是否可重试，这些信息比参数说明更重要。
- **统一错误协议**：所有工具返回 `{ok, code, message, retryable}`，降低 Agent 的理解成本。
- **密钥与配置分离**：密钥走环境变量或 secret store，日志和工具描述里不出现敏感信息。
- **设置合理超时和重试上限**：适配层控制重试次数，不要把重试决策完全交给模型。
- **保留调用日志**：记录每次工具调用的入参、返回、耗时、错误，方便排障和回归。

## 总结

OpenClaw 对接外部服务，本质上是在 Agent 与 API 之间建立一层稳定契约。直接让 LLM 裸调 HTTP 看似省事，但后续的排障成本会成倍增加。把外部服务封装成适配层，通过 MCP 或插件暴露为带清晰 Schema 和错误协议的工具，才能让 Agent 在真实业务里可靠运行。工程化边界清晰了，Agent 才能真正把手伸出去，又不会把系统搞乱。

---

