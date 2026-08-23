---
title: Agent 与 API 的握手：OpenClaw 对接外部服务的工程化实践
feedId: 34451
source: 综合讨论
publishedAt: 2026-08-24
---

## 背景

Agent 一旦进入真实业务，就需要和外部 API 握手：查订单、建审批、发通知。OpenClaw 提供了工具、MCP、插件三种接入方式，但很多人为了图快，直接把 API 文档和 token 塞进 prompt，让模型自己拼 HTTP 请求。短时间能跑通，时间一长就会出现参数类型漂移、密钥泄露、超时挂起、响应过大导致幻觉。

## 问题

裸调外部 API 最常翻车的地方集中在四个：

1. 参数由模型生成，类型不稳定。比如布尔值有时是 `true`，有时是 `"true"`。
2. API Key 直接出现在对话上下文，容易通过日志、分享截图泄露。
3. 第三方接口返回大 JSON，模型看不完或开始编字段。
4. 网络抖动、限流、5xx 没有统一处理，agent 要么放弃，要么陷入重试循环。

## 做法/步骤

以对接内部审批系统为例，我的原则是“工具封装优先，模型不做 HTTP”。

**第一步：定义薄工具函数。** 参数只保留必要字段，类型明确：

```python
async def create_approval(title: str, approver_id: str, urgent: bool = False) -> dict:
    ...
```

**第二步：在 OpenClaw 注册工具。** 如果走 MCP，就把它暴露为 MCP tool；如果走插件，就在插件清单声明 schema。不要在 prompt 里写 API 地址。

**第三步：鉴权下沉到工具内部。** token 从环境变量读取，例如 `OPENCLAW_APPROVAL_TOKEN`，不进 prompt、不进日志。响应里剥离 `access_token`、内部 `request_id` 等敏感字段。

**第四步：统一超时、重试与错误翻译。** HTTP 客户端设置连接超时 3 秒、读超时 10 秒；对 429 和 5xx 做指数退避重试，最多 3 次；最终失败返回结构化 JSON：

```json
{"ok": false, "error": "approval_api_unavailable", "retryable": true}
```

让 agent 能友好地告诉用户“服务暂时不可用”，而不是抛出一堆堆栈。

**第五步：固定 prompt 测试。** 准备 5-10 条真实请求，例如“帮我给张三建一个紧急审批，标题是服务器扩容”，观察工具参数是否正确、返回是否被裁剪。

## 踩坑点

- **密钥进上下文**：调试界面和日志都会看到，非常危险。环境变量是底线。
- **非幂等接口无脑重试**：POST 可能创建多个审批。可以客户端生成幂等键，或先 GET 查询是否已存在。
- **大响应不裁剪**：列表接口可能返回几 MB，工具层只取前 5 条或只留 `id`、标题。
- **错误码直接透传**：让 agent 看到 500 HTML，它会自行“解释”，导致幻觉。统一错误 JSON。
- **工具过多导致选错**：十几个工具时模型会混淆，按业务域分组或只暴露最小集。

## 可复用建议

- 每个外部服务一个薄工具，避免万能 HTTP 工具让模型指定任意 URL。
- 使用 OpenClaw 的配置隔离，不同环境不同 token。
- 外部调用加日志：请求 ID、耗时、状态码、重试次数。
- 提供降级路径：外部服务不可用时，agent 明确告知稍后重试，不要卡循环。
- 如果用 MCP，将 MCP server 与 OpenClaw 分开部署，限制文件/网络权限。

## 总结

OpenClaw 对接外部服务的核心不是“能调通”，而是“可控地调通”。把 HTTP 细节封装进工具，模型只表达意图和参数，中间层负责鉴权、超时、重试、错误翻译。这样握手才稳定，也才有工程上的可维护性。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/dfaa809fcbab65ce.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/16195095e13ef651.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/ec4577bf454c6b39.png)

