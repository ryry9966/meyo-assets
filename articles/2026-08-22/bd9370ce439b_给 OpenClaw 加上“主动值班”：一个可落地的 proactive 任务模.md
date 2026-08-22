---
title: 给 OpenClaw 加上“主动值班”：一个可落地的 proactive 任务模板
feedId: 34162
source: 综合讨论
publishedAt: 2026-08-22
---

## 背景

大多数 OpenClaw / Agent 实践还停留在“人在回路”的问答模式：你发指令，它执行，一轮结束。真正有用的自动化场景——监控、巡检、定时汇总、状态变化处理——需要 agent 主动发起。Proactive 并不是“模型自己突然想干活”，而是用工程手段给它一个触发器，让它在没有用户输入时按规则跑起来。

如果你已经在用 OpenClaw 做 MCP 工具调用、插件编排或自动化脚本，会发现一件事：模型调用工具的能力不差，但默认缺少“何时该跑一次”的机制。补齐这个机制，agent 就能从问答机变成值班员。

## 问题

直接让 agent“每小时自己看一下”会遇到三个现实问题：

1. **输出不稳定**：模型可能多解释、包 markdown、漏字段，下游没法直接消费。
2. **重复执行与误报**：没有状态记录，同一个异常会反复告警，甚至网络抖动就触发一轮骚扰。
3. **写操作风险高**：一旦 agent 能主动调用 delete、restart 等工具，很容易在判断失误时造成真实影响。

所以 proactive 任务不能把整个流程都交给模型自由发挥，应该拆成可控制的小段。

## 做法/步骤

以一个最小可落地任务为例：**每 30 分钟检查一个服务健康状态，异常时发通知并写日志**。

### 1. 定义任务契约

写一个 `task.md`，把角色、输入、允许工具、输出格式限制死：

```markdown
你是值班巡检 agent。读取状态文件 {{STATE_FILE}} 和目标地址 {{TARGET_URL}}。
只允许调用：fetch_status, send_notify, append_log。
只输出 JSON，不要代码块，不要解释：
{"action":"notify|log|none","reason":"...","severity":"low|high"}
```

### 2. 用系统触发器跑 headless 任务

不要依赖 OpenClaw 会话常驻。用 cron 或 systemd timer 每 30 分钟触发一次：

```bash
*/30 * * * * /usr/local/bin/openclaw run task.md --json > /tmp/proactive_last.json
```

如果你的 runner 不提供 `run` 子命令，就用 headless API 把 `task.md` 转成一次对话，关闭多轮闲聊，`temperature` 调低。

### 3. 工具白名单

`fetch_status` 可以用 shell 包装：

```bash
curl -o /dev/null -s -w '%{http_code} %{time_total}' --max-time 10 "$TARGET_URL"
```

`send_notify` 走企业微信 / Telegram webhook；`append_log` 追加到 `incidents.log`。如果已有 MCP server，直接把这些能力暴露成 MCP 工具，agent 只负责编排。

### 4. 状态去重

每次执行后写 `state.json`：

```json
{"last_alert_ts": 1710000000, "last_status": "down"}
```

任务开始时先读状态。如果上次告警在 30 分钟内且当前仍 down，就只 log 不 notify，避免重复轰炸。

### 5. 先跑影子模式

第一周只执行 `fetch_status` 和 `append_log`，不调用 `send_notify`。把每次“拟告警内容”写进 `dry_run.log`，人工复核后确认没有误报再放开真实通知。

## 踩坑点

- **JSON 输出不稳定**：模型偶尔会包 markdown 或加解释。解析失败时重试一次，并在 prompt 里反复强调“只输出 JSON，不要代码块”。
- **误报风暴**：网络抖动会触发连续告警。加冷却时间，并设置连续 N 次异常才发通知。
- **时区问题**：cron 常跑在 UTC，日志时间对不上。显式传 `TZ` 或统一使用带时区的 ISO8601。
- **自动执行写操作**：白名单之外的工具一律禁止。危险操作必须 dry-run，或者增加二次人工确认。
- **Token 成本**：不要让模型读大日志。先用 grep/awk 预处理，模型只接收摘要和字段。

## 可复用建议

把 proactive 任务固定成四段：**触发 - 判断 - 执行 - 汇报**。

- 触发器用 cron / systemd timer；
- 判断逻辑优先用规则或脚本，模型只做轻量决策；
- 执行走 MCP 或白名单工具；
- 汇报走 webhook、日志或消息通道。

所有主动任务必须带状态文件，否则去重、冷却、恢复通知都做不了。先做只读巡检、每日汇总这类低风险场景，再逐步放开写操作。给 agent 加一个 heartbeat：每次任务结束写时间戳，另一个监控脚本检查心跳，防止静默失败。

## 总结

Proactive 不是让模型一直在线，而是“低频触发器 + 受限决策 + 白名单工具 + 状态清理”。先从每 30 分钟一次的服务巡检开始，把误报压下来，再扩展到其他场景。稳定之后，它比问答式 agent 更像一个可靠的值班员。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/75f845b15c604404.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/9c0a210ba7665068.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/f4734391308cf5da.png)

