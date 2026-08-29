---
title: Agent 与 API 的握手：OpenClaw 怎么对接外部服务
feedId: 35207
source: 综合讨论
publishedAt: 2026-08-29
---

# Agent 与 API 的握手：OpenClaw 怎么对接外部服务

## 背景

OpenClaw 里的 Agent 通常会做两类事：从外部服务拿数据，或者触发外部动作。比如查工单、读 CRM、创建任务、发送通知。模型本身没有这些能力，需要通过工具、MCP 或插件把外部 API 暴露给 Agent。

很多实践者容易把“对接外部服务”理解为“给个 URL 和 token 就能调”，但实际工程里，真正耗时的是参数约束、超时、错误映射和上下文控制。

## 问题

如果直接把原始 API 扔给模型，常见问题包括：

- 参数填错、遗漏，模型根据错误反复重试；
- 接口超时导致 Agent 卡住；
- 返回体太大，撑爆上下文窗口；
- API key 出现在工具描述里泄露；
- 写操作没有确认，误触发不可逆动作。

这些问题不是模型能力问题，而是工具边界没有设计好。

## 做法/步骤

### 1. 选择对接路径

OpenClaw 里一般有三条路：

- **MCP**：适合标准工具、官方或社区已有 server 的场景；
- **HTTP Tool / OpenAPI 导入**：适合自有 REST API；
- **自定义插件**：适合需要本地状态、复杂鉴权或非 HTTP 协议。

只读查询优先 MCP 或 HTTP Tool；写操作一律走显式审批。

### 2. 写一个薄 adapter

不要直接暴露原始 API。用 adapter 做参数校验、字段裁剪、重试、错误映射。例如外部工单系统接口是 `GET /api/tickets?status=open&limit=10`，在 OpenClaw 中配置成工具 `search_tickets`，只暴露必要参数。

示意配置：

```json
{
  "name": "search_tickets",
  "description": "Search support tickets. Read-only. Returns at most 10 items.",
  "parameters": {
    "status": {"type": "string", "enum": ["open", "closed"]},
    "limit": {"type": "integer", "minimum": 1, "maximum": 10}
  },
  "timeout_ms": 8000,
  "retry": 2
}
```

### 3. 配置鉴权、超时与重试

`base_url`、`token` 放环境变量，不要写进工具描述。超时建议 8–15 秒；重试 1–2 次，带退避。所有外部调用带 `request_id`，方便排障。

### 4. 错误映射

把 HTTP 错误转成模型能理解的结构化反馈：

- `401` → 认证失败，联系管理员；
- `429` → 外部限流，稍后重试；
- `500` → 上游异常，已记录 request_id。

不要把原始堆栈或 HTML 错误页塞给模型。

### 5. 用 mock server 测试

用固定 JSON 的 mock server 验证工具 schema、字段裁剪和错误路径，再连真实服务。这样能避免调试时打爆外部接口。

## 踩坑点

1. **API key 泄露**：工具描述里写 token，模型可能在输出中复述出来。统一环境变量注入。
2. **超时缺失**：外部接口卡 30 秒，Agent 就卡 30 秒。设置合理超时和熔断。
3. **返回体膨胀**：有一次对接 CRM 搜索接口，adapter 没裁剪，返回整个 JSON 对象，导致上下文迅速耗尽。后来只保留 `id`、`name`、`owner`、`updated_at` 四个字段。
4. **参数太多**：嵌套过深的 schema 模型容易填错。尽量扁平化，参数不超过 5 个。
5. **分页被忽略**：查不到数据不一定没有数据，可能在第二页。要么限制返回前 20 条，要么提供 `cursor` 参数。
6. **时间格式混乱**：外部系统用 UTC，工具描述只写“最近”，会导致边界错误。统一用 ISO8601 并在 adapter 转换。

## 可复用建议

- 一个外部服务封装成一个明确工具，不要做一个“万能 API 调用”工具；
- 工具描述里写清只读/写操作、是否有副作用、返回数量上限；
- 外部调用统一加 `request_id`、超时、重试、日志；
- 写操作默认人工确认，测试环境提供 `dry_run` 参数；
- 优先用 MCP 管理工具生命周期，减少插件碎片；
- 监控外部调用成功率、P95 延迟和 token 消耗。

## 总结

OpenClaw 对接外部服务的核心不是“能调通”，而是把外部不确定性关进 adapter 的笼子。握手成功的标志是：模型知道何时调用、只传必要参数、失败有明确反馈、写操作有确认。这样 Agent 才能在真实业务里稳定扩展，而不是只在 demo 里跑通一次。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/92a931f91dddf325.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/3b3d63b531d258a3.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/01b7e07668f2e761.png)

