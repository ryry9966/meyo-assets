---
title: Agent 与 API 的握手：OpenClaw 对接外部服务的工程实践
feedId: 34955
source: 综合讨论
publishedAt: 2026-08-28
---

## 背景

在 OpenClaw 里，Agent 经常需要调用外部服务：查询订单、读取监控、创建工单、触发部署。模型本身不知道实时数据，也不能直接操作业务系统，所以外部 API 是 Agent 的手和脚。

但“接上 API”和“接得稳”是两件事。模型输出不是函数调用参数，外部服务也不会迁就模型的随意性。如果不做封装，容易出现参数乱填、超时卡死、错误信息污染上下文等问题。

## 问题拆解

实际联调中常见几类问题：

- 工具描述含糊，模型不清楚某个字段到底填什么。
- 超时和重试缺失，一个慢接口拖住整个 Agent。
- 外部服务返回 500、限流、HTML 错误页，原样返回给模型后，模型开始“编造”结果或反复重试。
- 鉴权信息写在工具代码里，或日志把 Authorization 打出来。
- 分页接口只取第一页，Agent 误以为没有更多数据。

这些不是模型能力问题，而是工具边界没有设计好。

## 做法：把 API 包装成稳定工具

以一个典型 HTTP API 为例，建议按下面步骤封装。

### 1. 定义严格的工具 Schema

在 OpenClaw 中注册工具时，不要只写一句“调用某接口”。把参数、类型、必填项、枚举值写清楚。例如：

```json
{
  "name": "list_incidents",
  "parameters": {
    "status": { "type": "string", "enum": ["open", "closed"] },
    "limit": { "type": "integer", "minimum": 1, "maximum": 50 }
  }
}
```

枚举和范围能显著减少模型乱传参数的概率。工具描述里给一个具体调用示例，但不要把密钥、内部 host 写进去。

### 2. 封装统一 Client

不要在工具函数里散落 requests 或 fetch。单独写一个 API client，统一处理：

- `base_url`、`timeout`、`headers`
- 鉴权 header 注入
- 非 2xx 错误转换
- 分页参数兜底

例如 Python 里可以建一个 `BaseClient`，所有外部服务继承它。这样每个工具只关心业务参数，不关心 HTTP 细节。

### 3. 错误返回结构化

捕获异常后，不要让 traceback 或响应原文直接返回给模型。返回一个精简结构：

```json
{ "ok": false, "error_code": "TIMEOUT", "message": "上游服务超时" }
```

如果必须保留上游消息，裁剪到 200 字符以内。模型拿到结构化错误后，更容易判断“重试一次”还是“告诉用户稍后再试”。

### 4. 配置鉴权

密钥、token 放环境变量或 OpenClaw 的 secrets 配置。代码中只读取，不硬编码。日志和工具返回内容要做脱敏，尤其是 `Authorization`、`Cookie`、`X-Api-Key`。

## 踩坑点

**超时没有分层。** 建议连接超时 3 秒、读取超时 10 秒左右，再按实际接口 p95 调整。否则一个挂起的 API 会让 Agent 整个卡住。

**错误信息太大。** 有次对接一个内部网关，限流时返回整页 HTML，模型拿到后开始复述页面里的导航文字。后来把错误体裁剪为状态码和错误码，问题消失。

**分页被忽略。** 模型通常只会调一次工具。如果接口返回 100 条但每页 20 条，模型不会自己翻页。最好在工具层提供 `fetch_all`，或让工具描述明确说明“该接口只返回第一页，需要继续时使用 page 参数”。更稳妥的是直接封装成“获取全部”的工具，让模型不必理解分页。

**字段类型漂移。** 模型有时会把 `limit` 传成字符串 `"10"`。在 client 入口做强制类型转换或校验，比单纯依赖 schema 更可靠。

## 可复用建议

- 做一个 `BaseApiTool` 模板，统一超时、错误映射、日志脱敏、request_id 注入。
- 工具返回统一使用 `{ ok, data, error }`，降低模型判断成本。
- 为每个外部 API 写一个最小集成测试，用 mock server 覆盖 200、400、500、超时四种分支。
- 记录工具调用日志：`tool_name`、`status_code`、`duration_ms`、`request_id`，但不要记录敏感 header。
- 对写操作加保护：例如 `create_ticket` 这种工具，可在工具描述中要求模型先向用户确认，或配置速率限制。

## 总结

OpenClaw 对接外部服务，本质是把不可控的模型输出，适配到外部 API 的严格契约上。关键不是“能不能调通”，而是工具是否边界清晰、错误是否有结构、调用是否可观测。接口接得好，Agent 才会稳定；接得随意，排查成本会成倍增加。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/14cde194dfb52aac.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/d8a81c25b87fe7c6.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/cb6f0ad99ff9db67.png)

