---
title: 从零搭建 AI 社区发帖机器人：基于 OpenClaw + MCP 的架构设计与踩坑实录
feedId: 32151
source: 综合讨论
publishedAt: 2026-08-08
---

## 背景

我们维护的开发者社区每天需要发布技术日报、问答精选和文档更新，重复性极高。早期靠人工搬运，每周编辑花费 6~8 小时。既低效又容易出错，一旦编辑休假内容就断档。团队决定做一个能动态编排、自主生成并发布的机器人，但不满足于简单脚本——希望它能结合 Agent 的推理做内容质量控制，具备多平台扩展能力，且内部审计可追踪。

技术选型用了 OpenClaw 作为任务编排层，MCP (Model Context Protocol) 作为工具接入标准，LLM 负责内容生产与审查。整体不需要外挂 UI，运行在服务器 sidecar 模式。

## 要解决的真实问题

- **多源内容聚合**：从 RSS、GitHub trending、内部 CMS 捞取候选素材
- **内容安全与风格控制**：生成帖必须符合社区规范，不能出现幻觉式外链、非授权引用
- **多社区发布**：Discourse、飞书文档、Reddit (read-only 测试)
- **反爬与限流**：各平台对自动化工具极其敏感，一不小心就封号
- **调试与可观测性**：每次发布的决策过程需留痕，便于人工复核

## 架构设计

整体分为四层：

```
数据采集层 -> 内容加工层 (LLM + 审查规则) -> 任务调度层 (OpenClaw Agent) -> 发布执行层 (MCP Servers)
```

### 1. 数据采集层
通过 Cron 定时调用内部 API 汇总素材，输出统一的 `source_items` JSON，包含标题、摘要、原始链接、来源类型、时间戳。不做大模型调用，减少 token 消耗。

### 2. 内容加工层
一个轻量级 Python 服务实现内容加工，具体流程：
- 调用 LLM 对候选素材排序和生成摘要
- 基于 pre-defined 规则过滤：去重、检查敏感词、黑名单域名
- 用模板生成 Markdown 草案，返回给 Agent 做二次校验

这里特别做了**双模型分层**：排序与摘要用便宜的 4-turbo，策略推理用能力更强的模型，但只对高价值候选帖启用。

### 3. 任务调度层 —— OpenClaw Agent
OpenClaw 作为分布式 Agent 运行时，负责串联加工层和发布层。定义一个 `daily_post` 的 workflow：

1. 调用 `fetch_sources` 工具（MCP）获得素材
2. 将素材传给加工层 API 得到草案
3. 人工审核步骤（可选）：发送草案到内部飞书群，`wait_for_approval` Action 挂起
4. 审核通过或超时自动通过，调用 `publish_to_community` 工具（MCP）发布
5. 将发布状态、链接、快照写入日志

这里有个关键设计：发布工具不直接暴露原始 HTTP 请求，而是封装为 MCP Server，内部处理重试、签名、cookie 维护。Agent 只关心 `publish` 成功或失败。

### 4. 发布执行层 —— MCP Servers
针对每个社区实现一个 MCP Server，例如 `discourse-mcp`：
- 基于 Discourse API，支持创建帖子、回复、上传图片
- 内置安全限速（2 次/分钟）和自动等待
- 遇到 429 自动退避，并将退避时长上报给 Agent，告知下次重试时间

对于没有标准 API 的社区，使用 Playwright 封装为 `browser-mcp`，但踩坑无数，后面会讲。

## 核心步骤

### 数据源到草案
加工层 API 使用 FastAPI 实现，接口接受 `source_items` 返回 `draft`。为了保证内容质量，强制要求：
- 每条源信息必须附带 `source_url`，否则丢弃
- 对所有外链做 HEAD 请求，确认可达性（超时 5s）
- 敏感词使用双数组 Trie 匹配，避免 LLM 遗漏

### Agent workflow 编排
在 OpenClaw 中，workflow 以 YAML 定义，支持工具调用、条件分支和人工卡点。我们的 `daily_post` workflow 片段：
```yaml
steps:
  - id: fetch
    tool: fetch_sources
  - id: draft
    action: call_http
    params:
      url: "http://content-processor/draft"
      method: POST
      body: "{{fetch.result}}"
  - id: review
    action: human_approval
    message: "请审核即将发布的帖子：{{draft.title}}"
    timeout: 3600
  - id: publish
    tool: discourse_publish
    params:
      title: "{{draft.title}}"
      content: "{{draft.content}}"
    condition: "review.approved == true"
```
每个步骤失败都有重试策略，最多 3 次，指数退避。

### 发布后的监控
Agent 完成后推送消息到内部监控频道，包含帖子链接、素材来源、模型决策日志、耗时。如果发布失败，通知 on-call 人员。

## 踩坑实录

### 1. MCP 工具返回大体积 JSON 导致 Agent 超时
早期 fetch_sources 直接把完整 HTML 摘要嵌入 JSON，一次返回 3MB+。OpenClaw 的 gRPC 消息大小限制默认 4MB，加上序列化开销触发超时。解决办法：只传结构化字段，原始内容存储本地 S3，工具返回 presigned URL。

### 2. Discourse 的 CSRF Token 失效
Discourse API 要求先 GET 获取 CSRF token 再 POST。我们封装 MCP 时误用了固定 token，导致过一段时间就 403。正确做法是每次会话维护一个 token，在 403 时自动刷新。关键是要在 MCP Server 中做 connection pooling，避免每个请求重新认证。

### 3. 飞书文档发布时表格格式不兼容
飞书文档 API 对 Markdown 表格支持极差，经常错位。最后妥协：事先定义好支持的组件列表，如果 LLM 生成了表格，自动转为图片嵌入（通过 headless chrome 渲染截图），虽然增加 2~3s 延迟，但格式无误。

### 4. 浏览器自动化发布的验证码地狱
尝试用 Playwright MCP 发帖到某个无 API 的论坛，但一登录就触发 reCAPTCHA。尝试用 2captcha 解决，但成功率只有 60%，且单次成本太高。最终放弃该社区，等官方开放 API。教训：对强制人机验证的平台，自动化 ROI 往往为负，除非业务价值巨大。

### 5. LLM 幻觉引用
草案中偶尔出现虚假的论文链接，看起来非常真。我们加了一个后处理步骤：用正则提取所有 URL，逐一 HEAD，返回非 2XX 的直接剔除链接，并在日志中标红。虽然不能完全杜绝，但生产环境从未再漏过。

## 可复用建议

- **MCP 工具设计遵循“单一职责、幂等性”**：比如 discourse_publish 只负责发布，内容校验放在 Agent 侧，工具不做复杂业务决策。
- **分层解耦，不要把所有逻辑塞进 Agent**：加工层独立部署、独立监控，Agent 只做协调，这样换模型、换策略时不影响其他部分。
- **给所有外部调用加可观测性**：为每个 MCP 调用加 trace，记录 latency、status、payload digest，出问题时快速定位。
- **审核卡点不能省**：即使全自动，也保留人工抽样介入的通道。我们现在是 70% 自动通过，30% 抽样在线审核，平衡效率与风险。
- **部署清单化**：把 MCP server 做成 Docker 镜像，通过 compose 统一管理，环境变量注入社区密钥。Agent workflow 用版本化管理，每次变更必须走 CI。

## 总结

从零搭建发帖机器人并不是简单的 `requests.post()`。真正的挑战在于把分散的工具、脆弱的平台接口和强 AI 能力组合成稳定耐用的工程管线。采用 OpenClaw + MCP 的架构让我们能模块化地增删社区、优雅地处理故障，也让团队从发帖苦力变成管线维护者。目前系统稳定运行 4 个月，日均发帖 3~5 篇，人工介入时间减少 85%。后续打算加入 A/B 测试框架，对比不同 LLM 生成内容在社区中的互动数据，形成正向飞轮。

希望这些实在的坑和设计思路能给同样在搭内部效率工具的同行一些参考。

---

