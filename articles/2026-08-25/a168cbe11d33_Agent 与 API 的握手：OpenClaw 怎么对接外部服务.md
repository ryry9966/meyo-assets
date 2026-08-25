---
title: Agent 与 API 的握手：OpenClaw 怎么对接外部服务
feedId: 34686
source: 综合讨论
publishedAt: 2026-08-25
---

## 背景

在 OpenClaw 里，Agent 要接外部服务，通常有两条路：一是让模型直接生成 HTTP 请求或执行代码，二是把外部服务封装成 Tool 或 MCP server。工程实践上，后者稳定得多。原因是 LLM 适合表达“要做什么”，但不适合可靠地处理鉴权、重试、超时、限流和错误恢复。

把 API 握手交给确定性工程层，Agent 只需要发起工具调用，这是更可控的边界。

## 问题

如果不做封装，直接让 Agent 面对外部 API，通常会出现几类麻烦：

- 鉴权信息被写进 prompt 或工具描述，泄露风险高；
- API 超时设置不合理，Agent 卡住，用户误以为服务挂了；
- 第三方返回大 JSON 或错误 HTML，上下文被撑爆；
- 一次失败后模型反复重试，触发外部限流；
- 原始错误信息直接回传，模型被错误页带偏，后续推理混乱。

这些问题不是模型能力问题，而是边界问题：传输、鉴权、错误映射这些事，应该由确定性的工程代码完成，而不是让概率模型自由发挥。

## 做法/步骤

1. **定义工具边界**  
   把外部服务抽象成一个明确动作，例如查询订单、创建工单、发送通知。不要在工具描述里写完整 HTTP 细节，只说明“做什么、需要什么参数、返回什么”。

2. **写死输入/输出 JSON Schema**  
   OpenClaw 注册工具时要求输入 schema。建议设置 `additionalProperties: false`，对容易被模型乱填的字段用 `enum` 限制。输出字段也只保留 Agent 决策需要的内容。

3. **实现 handler**  
   在 handler 内部完成这些事：
   - 从环境变量读取 API key，不硬编码；
   - 设置合理超时，一般 5–10 秒；
   - 仅对幂等请求做有限重试，退避 1s/2s/4s；
   - 把 401、403、429、5xx 统一转成简短错误码，不把 HTML 或堆栈回传；
   - 对响应做裁剪，只返回关键字段。

一个极简 schema 示例：

```yaml
name: get_order_status
description: 查询订单状态
input_schema:
  type: object
  properties:
    order_id:
      type: string
      description: 订单号
  required: [order_id]
  additionalProperties: false
```

4. **注册到 OpenClaw**  
   如果外部服务已有 MCP server，优先接 MCP；没有的话，再用插件或自定义 Tool 方式注册。优先复用，减少自己维护传输层代码。

5. **干跑测试**  
   先用固定输入直接调用 handler，不经过模型，确认能正常返回。再让 Agent 调度，观察工具调用是否稳定。

## 踩坑点

- **密钥管理**：API key 不要写进工具描述或模型上下文，统一走环境变量，并限制该 key 的权限范围。
- **超时与重试**：GET 查询类可以重试；创建类 POST 要谨慎，重复提交可能产生脏数据。
- **响应体积**：列表接口可能一次返回几 MB，必须截断、分页或只取摘要字段，否则上下文会迅速膨胀。
- **错误信息**：外部 API 的错误页可能是 HTML 或堆栈，直接回传会让模型产生错误推理。统一改成简短、可操作的错误信息。
- **Schema 太宽**：模型会生成不存在的参数。用 `required`、`enum`、`additionalProperties: false` 收紧输入。
- **限流**：多个 Agent 并发调用同一外部 API，容易触发限流。可在 handler 内做简单队列或指数退避。
- **日志**：记录 tool call id、外部 API 状态码、耗时、重试次数。否则排障时只能靠猜。

## 可复用建议

- 封装统一的 HTTP client，内置超时、重试、错误映射和日志。
- 配置与代码分离：`base_url`、`timeout`、`api_key` 全部走环境变量。
- 对外部 API 做一层防腐，不让第三方原始数据格式直接进入 Agent 上下文。
- 优先复用 MCP server，避免每个项目重复造轮子。
- 给每个工具写最小单测，至少覆盖 200、400、401、429、超时五种情况。

## 总结

OpenClaw 对接外部服务，核心不是让 Agent 学会调 API，而是把 API 握手过程工程化。Schema 管住输入，handler 管住传输和错误，MCP/插件管住复用。做到这三点，Agent 调用外部服务会从“碰运气”变成“可预期”。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/401475f37daad7d5.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/b6f8c98465584398.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/e0dbb7cbd5a321d2.png)

