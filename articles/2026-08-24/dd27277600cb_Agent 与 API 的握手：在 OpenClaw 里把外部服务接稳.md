---
title: Agent 与 API 的握手：在 OpenClaw 里把外部服务接稳
feedId: 34489
source: 综合讨论
publishedAt: 2026-08-24
---

## 背景

OpenClaw 的 Agent 大多数时候不是孤立运行，需要查询业务数据、触发审批、写入工单等。很多团队已经有一套现成 REST API。让 Agent 直接生成 HTTP 请求、拼 URL、塞 header，表面上省事，实际上联调时问题不断：模型容易把 query 参数写到 path 里，鉴权头被带进日志，错误响应被当成业务数据继续推理。我们最后采用的方案是：把外部 API 封装成 OpenClaw 工具，模型只负责选择工具和填参数，HTTP 细节放在工具层处理。

## 问题

直接让模型发 HTTP 请求有几个典型问题：
- 参数格式不稳定，日期、枚举、嵌套对象容易错。
- 鉴权信息难以安全注入，模型甚至可能自创 header。
- 响应体积不可控，一次列表查询可能返回几万 token，污染上下文。
- 网络超时、限流等异常如果没有结构化处理，模型会原地重试，浪费调用预算。

## 做法/步骤

我们对接的外部服务包括订单查询、库存查询、创建工单。重点不是让模型学会 HTTP，而是把 API 变成“函数”。

1. **圈定工具边界**  
   不把整本 API 文档丢给模型。每个工具只做一个动作，例如 `get_order_status(order_id)`。只读优先，写操作单独审批。

2. **选择接入方式**  
   有 OpenAPI 规范的项目可以直接生成 MCP server；没有规范就手写一个薄 adapter。OpenClaw 侧注册为工具，例如：
   ```json
   {
     "name": "get_order_status",
     "description": "Query order status by order ID",
     "parameters": {
       "type": "object",
       "properties": {
         "order_id": { "type": "string", "description": "Order ID, 18-digit" }
       },
       "required": ["order_id"]
     }
   }
   ```

3. **封装 HTTP 调用**  
   - 鉴权放在环境变量或密钥管理，不写入工具描述或模型上下文。
   - 超时设短，GET 通常 5 秒，POST 视业务 8-10 秒。
   - 幂等 GET 可以重试一次，非幂等写操作禁止自动重试。

4. **裁剪响应**  
   工具层只返回关键字段，例如订单状态、更新时间、异常码。统一包一层：
   ```json
   { "ok": true, "data": { "status": "SHIPPED" } }
   ```
   出错时：
   ```json
   { "ok": false, "error_code": "RATE_LIMITED", "retry_after": 2 }
   ```

5. **联调与监控**  
   用 mock server 先跑通参数，再接真实 API。日志里记录 request_id、状态码、耗时，不记录 Authorization 和请求体。

## 踩坑点

- **把 OpenAPI 全量导入**：模型会迷失在几十个接口里，选工具准确率下降。按业务动作裁剪。
- **让模型自己处理分页**：最好在工具内完成分页循环，返回汇总结果。
- **错误信息直接抛给模型**：模型看到一长串 HTML 错误页，可能会尝试“修复” URL 再试，形成重试风暴。错误必须结构化、可理解。
- **参数描述太模糊**：只写 `order_id: string` 不等于告诉模型它是 18 位数字还是 UUID。描述里给示例。
- **忽略时区**：日期时间参数如果不说明时区，查询边界会漂移。统一用 UTC 或带 offset。

## 可复用建议

- 统一做一个轻量 gateway，OpenClaw 只对接 gateway，由 gateway 负责鉴权、限流、日志、响应裁剪。外部服务变更不影响工具定义。
- 所有工具返回固定结构 `{ok, data, error}`，模型容易判断成功/失败。
- 写操作加确认，或限制每天调用次数，避免 Agent 误触发。
- 监控工具调用成功率、平均耗时、错误码分布，联调期能快速定位是参数问题还是服务问题。
- 响应大小设硬上限，超过就截断并标注 `truncated`，防止上下文爆炸。

## 总结

OpenClaw 对接外部服务，本质是把自然语言意图转成结构化工具调用。工程重点不在提示词，而在工具契约、响应裁剪和异常处理。把 HTTP 细节关进工具层，模型只做选择和填参，整个链路才会稳定可维护。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/1d3460dd24f97c97.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/3fa1a80ace6eeec4.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/0019509f8610956e.png)

