---
title: 让 AI 助手主动出击：用 OpenClaw 构建 Proactive Agent 的工程实践
feedId: 31980
source: 综合讨论
publishedAt: 2026-08-07
---

## 背景

目前大多数 AI 助手仍为被动响应式设计——你必须先发出一条消息，它才会给出回应。但在自动化运营、开发协作、运维监控等场景中，有大量的工作更适合由 agent **主动发起**，例如：定期检查代码仓库新 issue 并生成摘要、监控服务器异常并推送告警、提前提醒日程冲突。

OpenClaw 作为一个可编排、可扩展的 AI agent 平台，原生支持 **定时触发器**、**Webhook 触发** 以及通过 **MCP (Model Context Protocol)** 连接外部服务，这让我们可以用工程化的方式构建 **Proactive Agent**，让助手在你开口之前就把事情办了。

本文以一个**主动式 GitHub 仓库管家**为例，完整记录从设计、实现、踩坑到复用的工程经验。

## 问题拆解

要构建一个“不等你开口”的 agent，需要解决几个核心问题：

1. **触发机制**：谁来发起任务？在没有用户消息的情况下如何唤醒 agent？
2. **上下文获取**：主动任务通常需要实时外部数据，agent 如何安全、高效地拿到这些数据？
3. **状态管理**：如何避免重复处理、重复通知？
4. **人机交互平衡**：如何既保持主动，又不造成骚扰？

针对这些问题，我们的方案是：**OpenClaw 定时触发 + MCP 工具调用 + 去重状态存储 + 合并通知**。

## 实现步骤

### 1. 编写一个 MCP Server（GitHub 数据层）

首先需要让 agent 有获取 GitHub 仓库新 issue 的能力。我们用 MCP 协议暴露一个简单的工具 `get-recent-issues`，内部调用 GitHub REST API。

关键逻辑（Node.js 示例，MCP server 基于 `@modelcontextprotocol/sdk`）：

```ts
server.setRequestHandler(CallToolRequestSchema, async (request) => {
  if (request.params.name === "get-recent-issues") {
    const { owner, repo, since } = request.params.arguments;
    const url = `https://api.github.com/repos/${owner}/${repo}/issues?since=${since}&state=all`;
    const resp = await fetch(url, {
      headers: { Authorization: `Bearer ${GITHUB_TOKEN}` },
    });
    const issues = await resp.json();
    return {
      content: [{ type: "text", text: JSON.stringify(issues) }],
    };
  }
});
```

部署后，在 OpenClaw 的 MCP 配置中注册该 server，让 agent 可以调用此工具。

### 2. 在 OpenClaw 中创建 Agent 并配置定时触发器

创建一个专用 agent，system prompt 中明确其职责：“你是一个仓库管家，每次被唤醒时检查指定 GitHub 仓库在最近 10 分钟内的新 issue，分析内容并生成一份简洁的摘要报告。”

关键配置部分是**定时触发器**。OpenClaw 支持 cron 表达式定义周期性触发。我们在 agent 配置中添加：

```yaml
triggers:
  - type: schedule
    cron: "*/15 * * * *"
    message: "请立即检查 GitHub 仓库 my-org/my-project 在最近 15 分钟内的新 issue，并生成摘要。"
```

每 15 分钟，OpenClaw 会自动向这个 agent 发送一条包含以上消息的系统唤醒指令，agent 随即进入执行链路。

### 3. Agent 执行与通知推送

被唤醒后，agent 会调用 MCP 工具 `get-recent-issues`，参数 `since` 设置为当前时间减 15 分钟的 ISO 字符串。获取到 issue 列表后，agent 用 LLM 分析标题、正文、标签，生成结构化的摘要，例如：

```
#my-project 近期 Issue 动态 (2024-12-10 14:15)
- [bug] #234 登录页面崩溃 @zhangsan
- [feature] #235 增加暗色模式 @lisi
共 2 个新 Issue，暂无需立即处理的安全问题。
```

然后 agent 调用通知插件（如企业微信、Slack webhook）将摘要发送到指定频道。

### 4. 去重与状态管理

为了避免多次运行时重复报告同一个 issue，我们在 agent 中引入一个简单的“已处理 issue 集合”。实际实现上，我们利用外部 KV 存储（Redis）保存最近一次处理的 issue ID 和时间窗口。

在 `get-recent-issues` 工具中增加过滤参数 `exclude_ids`，由 agent 传入上次已处理过的 ID 列表。或者也可以在 agent 逻辑中，获取到结果后先过滤再生成摘要。为了保持无状态 agent 的简洁性，我们将去重逻辑放在 MCP server 端，server 维护一个简单的 LRU 内存缓存（重启丢失但可接受），记录最近一次成功推送的 issue ID 集合。

## 踩坑记录

**1. 触发器与 Agent 生命周期不一致**  
如果 agent 在无状态 serverless 环境中运行，每次触发都可能是一个新的实例，无法在本地内存中保存状态。因此去重必须依赖外部存储（Redis、文件、数据库）。我们一开始用内存缓存，导致每次重启后重复推送所有 issue，非常烦人。

**2. API 限流与重试**  
GitHub API 对匿名或未认证请求非常苛刻。即便带了 token，短时间内频繁请求也可能触发二级限流。我们的解决方式是：cron 间隔不低于 10 分钟，并在 MCP server 中增加简单的重试逻辑（最多 3 次，指数退避）。同时，对 `get-recent-issues` 返回的结果进行裁剪，只取关键字段，减少请求次数。

**3. 上下文过长**  
如果某个仓库突然收到大量 issue（例如机器人批量创建），一次性丢给 LLM 可能导致 token 超限。我们通过工具参数限制单次最大返回数量（如 10 条），并对超出的部分在摘要中注明“已截断，请手动查看”。

**4. 通知轰炸**  
初期版本中，我们让 agent 每发现一个新 issue 就发一条消息，结果频道被刷屏。后来改为每轮运行只生成一条合并摘要，并设置“仅当有新 issue 时才发送”的逻辑，大大改善了用户体验。

**5. 权限最小化**  
MCP server 直接调用 GitHub API，token 需要严格限制作用域（只读 issues 权限）。建议使用环境变量注入，并通过 OpenClaw 的 secret 管理功能托管 token。

## 可复用建议

- **将主动监控工具抽象为 MCP 标准接口**：例如 `query-since` 风格的工具，可以轻松替换后端（GitHub → GitLab → Jira）。
- **使用 OpenClaw 的 Skill/Template 功能**：把整个配置和 prompt 封装成可复用的“仓库管家 Skill”，其他团队只需修改仓库名和通知地址即可启用。
- **构建“礼貌”的 Agent**：主动不等于打扰。设计明确的触发频次上限、静默时段（如夜间不推送）、合并窗口，让用户感受到 assistant 而非 spammer。
- **增加手动入口**：保留一条“立即检查”的指令，让用户在等待主动推送前也能自行触发。
- **日志与可观测性**：为定时任务配置执行日志，记录每次唤醒是否成功、工具调用耗时、是否推送消息。这对排障至关重要。

## 总结

Proactive Agent 并不是简单的 cron job + LLM，它需要在**触发策略、状态一致性、API 容错、人机交互**这几个维度做细致的工程化处理。通过 OpenClaw 的定时触发、MCP 工具集成以及插件化的通知输出，我们可以把“主动服务”落地为可靠的生产级能力。

将 AI 助手从“等待指令”升级为“主动感知”，是 agent 走向真正有用的关键一步。希望本文的实践能为你打开一扇窗，让自动化里多一些智能，少一些手动。

---

