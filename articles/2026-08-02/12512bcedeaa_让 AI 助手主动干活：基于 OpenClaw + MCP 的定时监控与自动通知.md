---
title: 让 AI 助手主动干活：基于 OpenClaw + MCP 的定时监控与自动通知实践
feedId: 31289
source: 综合讨论
publishedAt: 2026-08-02
---

## 背景

“等用户开口”的 AI 助手，本质上还是高级的查询接口。而真正融入工作流的助手，应该具备 **proactive** 能力：在条件满足时主动触发动作，把结果直接推送到人和系统面前。

OpenClaw 社区一直强调 Agent 的自动化闭环，但很多实践仍停留在 “用户 @bot → 助手响应” 的被动模式。本文以一次真实的工程需求为例，记录如何让 OpenClaw 助手主动监控外部文档网站的变化，并在检测到更新后自动生成摘要、通过 Webhook 通知团队。全程基于 OpenClaw 内置的定时器、自定义 MCP 工具和少量胶水代码，实现一套可复用的 proactive 实践。

## 需求拆解

我们维护着一个开源项目，其上游 SDK 的更新日志页面变动频繁，但团队成员经常错过关键变更。理想方案是：助手每天定时拉取该页面，对比上一次的内容，如果发现有意义的差异，就生成一句话摘要，推送到企业微信群。

实现这个需求，助手需要具备三项主动能力：
1. **时间驱动**：无需用户交互，按 cron 表达式自动触发。
2. **状态记忆**：能记住上一次检查的状态，以此判断是否有变化。
3. **输出推送**：将摘要发送到外部系统（企业微信机器人），而不只是回复在对话窗口。

## 技术选型与架构

整体结构如下：

- **OpenClaw Agent**：作为主控，拥有一个定时 trigger 配置。
- **自定义 MCP Server**：提供 `fetch_page`、`diff_summarize`、`send_webhook` 三个工具，供 Agent 调用。
- **外部存储**：使用本地 JSON 文件保存上次页面哈希值与摘要时间戳（避免引入数据库依赖，适合小规模任务）。

选择 MCP 工具而非插件，是因为这些操作与外部 HTTP 调用、文件 IO 强相关，MCP 隔离度和可调试性更好。同时，MCP 工具可以在别的 Agent 中复用。

架构示意：
```
[Cron trigger in OpenClaw config]
         │
         ▼
[Agent 接收定时事件]
         │
         ▼ (invoke MCP tools)
  ┌──────────────────────┐
  │ fetch_page           │ → HTTP GET changelog
  │ diff_summarize       │ → 对比哈希, LLM 摘要
  │ send_webhook         │ → POST 到企微机器人
  └──────────────────────┘
         │
         ▼
[更新本地状态文件]
```

## 实现步骤

### 1. 配置 OpenClaw 定时触发器

在 OpenClaw 的 `agent` 配置中加入 `triggers`：

```yaml
triggers:
  - type: cron
    schedule: "0 10 * * *"          # 每天上午10点
    task: "check_upstream_changelog"
    params:
      url: "https://example.com/changelog"
      state_path: "/data/changelog_state.json"
      webhook_url: "https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=XXX"
```

OpenClaw 会在这个时间点生成一个内部事件，自动唤醒 Agent 并传入 `task` 与 `params`。Agent 需要能处理这个事件并调用工具。

### 2. 编写 MCP 工具

创建三个工具，使用 Python 的 `mcp` SDK 实现（仅给出核心逻辑，省略完整服务注册）：

**fetch_page**  
直接请求 URL，返回页面 body 文本和当前内容的 SHA256 哈希。注意设置合理的超时与 User‑Agent，避免被反爬。

**diff_summarize**  
输入当前哈希和上次存储的哈希（从 state 文件读取）。若相同则返回 `no_change: true`；否则，将新的页面内容切片后调用 LLM（可复用 Agent 自身的 LLM 能力，或直接使用 OpenAI API）生成不超过 100 字的英文摘要。返回摘要文本和新的哈希。

**send_webhook**  
接收 Markdown 格式的消息，调用企业微信 Webhook。注意加入异常重试和响应状态码检查。

关键代码片段：
```python
@mcp.tool()
async def diff_summarize(new_content: str, new_hash: str, old_hash: str) -> dict:
    if new_hash == old_hash:
        return {"no_change": True, "summary": "", "hash": old_hash}
    # 使用 LLM 总结新内容
    prompt = f"Summarize the following changelog in English within 100 words:\n\n{new_content[:4000]}"
    summary = await llm_chat(prompt)
    return {"no_change": False, "summary": summary.strip(), "hash": new_hash}
```

### 3. Agent 行为编排

Agent 收到 `check_upstream_changelog` 事件后，按序调用：
1. `fetch_page(url)` → 得到 `content` 和 `hash`
2. 读取本地 state 文件，获取 `old_hash`
3. `diff_summarize(content, hash, old_hash)` → 若 `no_change` 则结束
4. `send_webhook` 将摘要和链接发送到群
5. 将新 hash 和时间写回 state 文件

所有调用在 Agent 内部通过 function call 完成，无需硬编码流程，Agent 可以灵活处理中间异常（例如网络失败重试一次，或切换备用 URL）。

## 踩坑记录

- **状态文件并发访问**：当 agent 可能多实例部署时，本地 JSON 存在写入冲突风险。我们的场景是单 Agent 实例，如果未来扩展，需要改用 Redis 或数据库加锁。初期不要过度设计。
- **页面内容变化噪声**：很多页面包含动态时间戳或广告，导致哈希每天变化但实际内容未更新。解决方法是先用规则清洗无关部分（例如去掉 footer 的时间），再计算哈希。
- **LLM 摘要的稳定性**：温度设为 0，并限定输出长度。曾经遇到 LLM 偶尔编造不存在的功能描述，通过在 prompt 中强调 “Only describe changes explicitly mentioned” 改善了幻觉。
- **Webhook 限流**：企业微信机器人有频率限制（20条/分钟以内），当前任务只发一条，但若后续扩展多源监控，需要加入队列和间隔控制。
- **定时触发不精确**：OpenClaw 的 cron 触发器可能因 Agent 冷启动延迟 1‑2 分钟，对日常巡检无影响，但如果用于准实时场景需注意。

## 可复用建议

1. **Proactive 能力应退化为事件驱动架构**：把定时任务看作一种事件源，工具集作为可替换的执行单元。这样做的好处是，你可以随时把“定时检查”换成“webhook 触发”，而不必修改工具和 Agent 逻辑。
2. **工具设计遵循单一职责**：`fetch_page` 不做总结，`diff_summarize` 不发送消息。组合能力交给 Agent 的计划模块，后期调整流程更容易。
3. **始终实现幂等性**：在 `diff_summarize` 中，如果哈希相同，直接返回 `no_change`，确保即使 Agent 被重复触发，也不会发送重复通知。状态更新的写入也应设计为覆盖式（原子写文件），避免残留脏数据。
4. **监控 proactive 任务的健康度**：我们额外增加了一个心跳监控：如果任务在预定时间后 30 分钟内未执行，会有一条告警发给自己。这种 meta 监控能避免“助手沉默了但无人知晓”的情况。
5. **向团队透明化**：在通知消息中附加状态文件路径和日志链接，方便成员排查为什么某天没有摘要。

## 总结

AI 助手的 proactive 能力本质上是一套可靠的 **感知‑决策‑执行** 回路。OpenClaw 提供了灵活的 trigger 和 MCP 扩展机制，使我们可以将“主动”封装进清晰的工具和事件流程中，而非仅仅依赖 prompt 的黑魔法。这次实践后，我们的上游变更不再被动追踪，团队信息同步效率明显提升。更重要的是，这套定时检查 + 摘要推送的模式，可以低成本复制到监控竞品动态、感知政策更新等场景。

未来还可以进一步引入更细粒度的 diff（比如只关注某个 DOM 节点的变化），或者将摘要自动翻译为中文推送给不同群组。Proactive 的想象力受限于事件源和执行力，而工程上的克制设计能让它可持续运行。

---

