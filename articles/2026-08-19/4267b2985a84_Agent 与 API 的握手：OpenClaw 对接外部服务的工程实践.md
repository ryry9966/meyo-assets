---
title: Agent 与 API 的握手：OpenClaw 对接外部服务的工程实践
feedId: 33780
source: 综合讨论
publishedAt: 2026-08-19
---

## 背景

Agent 真正产生生产力，不是靠“会聊天”，而是能稳定地触达外部系统：查工单、发告警、建分支、拉报表。OpenClaw 作为可扩展的 Agent 运行时，提供了 Tool、Plugin 和 MCP 等机制，让模型不再直接裸调 HTTP，而是通过一层受控的适配层与外部 API 握手。

但“能调通”和“能上线”是两回事。直接让模型拼 URL、塞 token、读原始响应，很快会遇到鉴权散落、错误不可控、超时无感、上下文爆炸等问题。本文基于实际对接经历，整理一套 OpenClaw 对接外部服务的做法。

## 问题

典型的失败模式有四种：

1. **把 API 当网页喂给模型**：原始 JSON 又长又吵，模型容易读错字段，token 消耗大。
2. **鉴权逻辑混在提示词里**：key 泄露、环境切换困难、无法审计。
3. **错误处理缺失**：外部服务 429/500 时，Agent 直接把异常当事实返回。
4. **没有边界**：一个工具可以写库、删资源、发通知，权限过大。

这些问题的本质是：Agent 需要的是**结构化、可预期、可观测**的工具，而不是一个任意的 HTTP 出口。

## 做法/步骤

以一个实际场景为例：让 OpenClaw 查询内部 GitLab 仓库的 Issue 列表，并汇总近 7 天未关闭的高优缺陷。

### 1. 定义工具边界与 Schema

先明确工具名、入参、出参。不要让模型决定一切。

```json
{
  "name": "list_project_issues",
  "description": "List issues in a GitLab project within a date range",
  "parameters": {
    "type": "object",
    "properties": {
      "project_id": { "type": "string" },
      "since_days": { "type": "integer", "default": 7 },
      "labels": { "type": "array", "items": { "type": "string" } }
    },
    "required": ["project_id"]
  }
}
```

出参不要直接返回 GitLab 原始数组，而是压缩成精简对象：`id, title, state, priority, web_url, updated_at`。这样模型接收到的信息密度高，且不会超上下文。

### 2. 实现 Adapter

在 OpenClaw 插件或 MCP server 中实现工具逻辑。核心是：**入参校验 → 组装请求 → 超时控制 → 错误映射 → 裁剪响应**。

```ts
export async function listProjectIssues(args) {
  const { project_id, since_days = 7, labels = [] } = args;
  const since = new Date(Date.now() - since_days * 86400_000).toISOString();
  const params = new URLSearchParams({ since, state: "opened" });
  labels.forEach(l => params.append("labels", l));

  const resp = await httpClient.get(`/projects/${encodeURIComponent(project_id)}/issues`, {
    params,
    timeout: 8000,
  });

  // 只保留必要字段，避免原始响应直接进入模型
  return resp.data.map(issue => ({
    id: issue.iid,
    title: issue.title,
    state: issue.state,
    priority: issue.labels?.find(l => l.startsWith("priority::")) ?? "none",
    url: issue.web_url,
  }));
}
```

### 3. 注册到 OpenClaw

如果走 MCP，将服务暴露为 MCP tool；如果走 Plugin，按 OpenClaw 的插件规范导出。关键是工具描述要写清楚**什么时候该用、什么时候不该用**，减少模型误调用。

### 4. 测试与观测

至少覆盖两类测试：

- **单测**：mock HTTP，验证参数拼接、错误映射、响应裁剪。
- **集成测试**：在测试环境跑通真实 API，观察超时与限流表现。

同时给每次调用打上 `trace_id`，方便在 OpenClaw 日志里跟踪“模型为什么调了这个工具、返回了什么、后续又做了什么”。

## 踩坑点

1. **超时不是越小越好**。外部 API 可能本身慢，比如 GitLab 查询大项目 Issue 要 3-5 秒。设 1 秒必挂。建议从 8 秒起步，按服务调整。
2. **分页不处理，数据必然漏**。GitLab 默认每页 20 条，如果不翻页，Agent 会得出“只有 20 个 Issue”的错误结论。Adapter 里要循环拉取，设定最大页数和总条数上限。
3. **错误不能直接抛给模型**。把 401 映射成“鉴权失败，请检查配置”，429 映射成“请求过于频繁，稍后再试”，500 映射成“服务暂不可用”。模型拿到结构化错误，才能给用户合理回复。
4. **不要返回 HTML 或超大 JSON**。一旦某个字段包含富文本或 base64，上下文会瞬间爆掉。Adapter 必须做字段白名单。
5. **权限收敛**。只暴露查询接口，写操作单独定义工具，并加确认步骤或人工审批。

## 可复用建议

- **统一 HTTP 封装**：一个带超时、重试、日志、trace_id 的 client，所有外部对接复用。
- **工具 Schema 即文档**：入参写清楚类型、默认值、边界，让模型少犯错。
- **做错误归一化**：对外部错误码建立映射表，统一返回 `{ ok: false, code, message }` 结构。
- **响应裁剪层**：每个 adapter 只返回模型需要的字段，不要“全量透传”。
- **提供 dry-run**：对危险操作，先让工具返回“将执行什么”，由用户确认后再真实调用。
- **监控调用**：记录每次工具调用的耗时、状态码、重试次数，便于定位是 Agent 问题还是外部服务问题。

## 总结

OpenClaw 对接外部服务，本质是给 Agent 装上一组**边界清晰、行为可控的“手”**。成功的对接不是让模型能调更多 API，而是让它只通过你设计好的窄接口，稳定地完成有限任务。把鉴权、超时、分页、错误映射这些脏活放在 Adapter 层，模型才能真正专注于决策和表达。工程化的目标永远是：**让 Agent 的每一次握手，都可预期、可复现、可回滚。**

---

