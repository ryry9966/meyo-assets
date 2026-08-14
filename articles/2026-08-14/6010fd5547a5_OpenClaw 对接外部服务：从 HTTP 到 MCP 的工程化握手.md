---
title: OpenClaw 对接外部服务：从 HTTP 到 MCP 的工程化握手
feedId: 33141
source: 综合讨论
publishedAt: 2026-08-14
---

# Agent 与 API 的握手：OpenClaw 怎么对接外部服务

## 背景

OpenClaw 里 Agent 能不能干活，很多时候不取决于模型推理能力，而取决于它能不能稳定地调用外部服务。查订单、写工单、读 CRM、触发部署，最终都要落到某个 HTTP API 或数据库上。

对接外部服务看起来简单：给个 endpoint、一个 token、描述一下参数。但实际跑起来，问题往往不在“调不通”，而在参数类型错、分页拉爆、超时挂起、鉴权泄漏、错误信息模型读不懂。下面按开源/自托管场景，讲一条可复现的对接路径。

## 问题

直接把一个原生 REST API 丢给 Agent，通常有三个坑：

1. **接口噪音大**：一个业务动作可能需要 3 个接口、5 个参数，模型容易漏传或错传。
2. **错误不可读**：外部返回 HTML、堆栈或 `{"code":500}`，模型不知道怎么修正。
3. **副作用不可控**：写操作没有重试边界，外部限流/超时直接拖死 Agent。

所以目标不是“让 OpenClaw 能请求外部服务”，而是把外部服务包成稳定、可复用、可观测的工具。

## 做法 / 步骤

以一个 CRM 服务对接为例。

**1. 先做工具抽象，不直接暴露原始 HTTP**

把外部服务拆成少量明确工具，比如 `search_customer`、`create_ticket`。每个工具只做一件事，内部组合多个接口。工具名和描述要写给模型看，避免“万能 API 工具”。

**2. 用 JSON Schema 严格约束参数**

工具入参不要用自然语言描述。要定义 `type`、`required`、`enum`、`default`。例如：

```json
{
  "name": "search_customer",
  "parameters": {
    "type": "object",
    "properties": {
      "keyword": {"type": "string", "minLength": 2},
      "limit": {"type": "integer", "default": 5, "maximum": 20}
    },
    "required": ["keyword"]
  }
}
```

这样模型传错类型时，OpenClaw 可以在调用前拦截并重新生成。

**3. 优先通过 MCP 接入**

把工具实现为一个 MCP server，OpenClaw 只作为 MCP client 使用。好处是工具可以独立开发、重启、测试，不污染 Agent 主进程。配置里只保留启动命令和环境变量，例如：

- 命令：`python -m crm_mcp`
- 环境变量：`CRM_TOKEN`、`CRM_BASE_URL`
- 超时：30s
- 重启策略：`on-failure`

Token 放在环境变量或 secret store，绝不写进工具描述或 prompt。

**4. 统一返回结构**

外部返回无论成功失败，都在 MCP 层包成：

```json
{
  "ok": true,
  "data": [],
  "error_code": null,
  "next_cursor": null
}
```

失败时返回可读错误码，比如 `CRM_RATE_LIMITED`、`CRM_BAD_KEYWORD`，并附带一句“建议模型修正动作”。这比把原始报错原样抛回给模型有效得多。

**5. 错误映射和重试**

- 4xx：不重试，把错误结构化返回给模型，让模型改参数。
- 429 / 5xx：做指数退避重试，最多 2 次。
- 超时：外部调用超时小于 Agent 等待上限，避免挂起。
- 写操作：只对幂等接口重试，非幂等接口必须返回错误让模型确认。

**6. 先 mock 后真联**

写一个本地 mock server，固定成功、参数错误、限流、超时四类响应。先在 mock 上验证 schema 和错误分支，再切真实服务。这样能避开很多“真服务不好造故障”的问题。

## 踩坑点

- **模型习惯于“猜参数”**：如果 schema 太宽，模型会传空值或错误枚举。把参数收紧到 `enum` + `default` 是避免返工最便宜的方式。
- **分页要返回游标，不要返回全量**：工具只返回第一页和 `next_cursor`，让 Agent 决定是否继续。一次拉全量会把上下文和内存打爆。
- **日志打码**：HTTP 请求 URL 里可能带 `?token=xxx`，记录日志前必须脱敏。建议统一在 adapter 层过滤 query 和 header。
- **MCP server 崩溃**：外部服务一更新，MCP server 可能抛异常退出。需要宿主做进程守护和重连，并且给 Agent 一个降级提示，比如“CRM 工具暂时不可用”。
- **不要在工具层塞太多上下文**：外部返回里的大段备注、日志、HTML 不要全量回传，只保留模型决策需要的字段。

## 可复用建议

1. **一个外部服务一个 adapter**：不要在 prompt 或 agent 逻辑里写死 API 细节，adapter 独立维护，版本可回滚。
2. **MCP 优先于插件脚本**：MCP 生命周期、schema、错误返回都更清晰，跨模型复用也容易。
3. **给每个请求加 `request_id`**：外部服务、MCP server、OpenClaw 日志都用同一个 ID，排障快很多。
4. **保留 golden response 做回归**：每次改 adapter，重放几条固定响应，确认 schema 没变。
5. **限制工具并发和频率**：外部服务脆弱时，限制 Agent 单次任务最大调用次数，避免复杂任务把 API 额度打光。

## 总结

OpenClaw 对接外部服务，不是把 API 地址丢给模型，而是把不可靠的外部世界包成可靠的工具契约。原则很简单：工具边界要窄、Schema 要严、错误要可读、重试要克制、日志要脱敏。做到这些，Agent 才能真正从“能说”变成“能办成事”。

---

