---
title: Agent 与 API 的握手：OpenClaw 对接外部服务的工程化实践
feedId: 31592
source: 综合讨论
publishedAt: 2026-08-04
---

Agent 的价值在于替人做决策，而决策依赖外部信息。OpenClaw 作为一个可编程的 agent 运行时，真正让它"有用"的不是聊天能力，而是能否稳定地调用外部服务——查库存、发通知、操作数据库、调内部系统。这篇文章不聊概念，只讲实践。

**你会遇到的三个问题**

第一，把 API 调用直接写死在 agent 逻辑里，密钥散落，换服务商要动核心代码。第二，以为"让 Agent 自己调 HTTP" 就行——模型生成 curl 看起来很灵活，但稳定性极差：参数格式错、鉴权暴露、超时无人处理。第三，忽略 4xx/5xx 和限流，agent 会把错误当作正常输出继续执行，直到用户发现数据错了。

**推荐做法：分层对接**

OpenClaw 对接外部服务，建议按场景分三层：

1. **MCP 方式**：适合标准化、可复用的服务（文件系统、GitHub、数据库）。MCP 把工具描述、入参 schema、鉴权都标准化，Agent 能感知工具并正确调用。
2. **Tool 函数方式**：适合私有或低频服务。注册一个普通函数，入参用严格 JSON Schema，出参用统一 envelope：`{ok, data, error}`。
3. **内置 HTTP action**：只用来快速验证连通性，别长期用。确定要接，就落到 1 或 2。

**一个具体例子：对接内部待办服务**

假设内部 API：`POST /api/todos`，需要 `Authorization` header，body 为 `{title, due_at}`。用 Tool 函数方式：

```python
@tool(schema={
    "title": {"type": "string", "description": "待办标题"},
    "due_at": {"type": "string", "description": "ISO 8601 截止时间，例如 2025-08-01T10:00:00Z"}
})
def create_todo(title: str, due_at: str) -> dict:
    resp = httpx.post(
        "https://internal.example.com/api/todos",
        headers={"Authorization": f"Bearer {os.environ['TODO_TOKEN']}"},
        json={"title": title, "due_at": due_at},
        timeout=10
    )
    if resp.status_code == 201:
        return {"ok": True, "data": resp.json()}
    return {"ok": False, "error": f"HTTP {resp.status_code}: {resp.text[:200]}"}
```

Agent 看到这个工具后，把用户的一句话翻译成参数，调用它，根据 `ok` 字段决定下一步。

**踩坑点整理**

- **超时独立设置**。对话流超时和 API 超时是两回事，工具内部必须设自己的 timeout，默认 10s，重试最多 2 次。
- **密钥不进对话**。用环境变量注入，别让密钥出现在 prompt、日志或 agent 可见的上下文里。
- **参数描述写清楚**。Agent 靠 description 决定怎么填参，写 `due_at` 不如写"ISO 8601 格式的截止时间"。
- **错误必须结构化**。不要返回 `failed`，要带状态码和截断的错误正文，agent 才能据此修复或告知用户。
- **重试要幂等**。发通知重试无妨，但创建订单这类操作，重试前先确认是否已成功，或携带 idempotency key。

**可复用建议**

1. 一个外部服务封装成一个独立 tool，单一职责；一个服务拆成多个 tool 也没问题。
2. 接入前先用 curl / Postman 验证请求和鉴权，别让 agent 去猜 API 文档。
3. 每个 tool 加日志：耗时、状态码、入参摘要（脱敏），排障全靠它。
4. 本地用 mock 服务（WireMock 或 FastAPI stub）跑通 agent 流程，再切真实 API。

**总结**

Agent 与 API 的握手，本质是职责划分：把不确定性留给 Agent，把确定性留给代码。模型负责理解意图、填充参数、判断下一步；代码负责鉴权、超时、重试和错误格式。OpenClaw 的意义在于把这条分界线画得足够清晰，剩下的，是你如何组织自己的工具集。

---

