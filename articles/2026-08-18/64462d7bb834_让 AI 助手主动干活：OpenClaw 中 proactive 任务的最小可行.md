---
title: 让 AI 助手主动干活：OpenClaw 中 proactive 任务的最小可行设计
feedId: 33667
source: 综合讨论
publishedAt: 2026-08-18
---

## 背景

大多数 AI 助手仍然是被动响应式的：你问一句，它答一句。但在真实开发/运维场景里，很多事需要“不用你开口”就有人盯着：凌晨 CI 挂了、接口延迟突然升高、关注的仓库发了新版本、证书还有三天过期。

这类需求用传统聊天式交互很难覆盖。你不可能每隔十分钟去问一次 Agent：“现在构建还好吗？” 更合理的方式是让 Agent 在后台按触发条件主动执行检查、通知甚至修复动作。这就是 proactive 能力。

OpenClaw 的用户大多熟悉 MCP、Agent 调度和插件自动化，天然适合做这类落地。但主动执行比被动回答危险得多：误报、打扰、权限过大、循环触发，任何一个都能让用户很快关掉所有主动能力。

## 问题

怎么给 OpenClaw 这类 Agent 加上 proactive 能力，同时避免失控？工程上至少要回答五个问题：

1. 触发源有哪些？
2. 什么条件下才执行？
3. 执行前要不要确认？
4. 权限边界在哪？
5. 重复执行、误报、循环触发怎么处理？

## 做法：把 proactive 拆成四段

不建议直接写一个“智能定时任务”一把梭。更可控的方式是把每条 proactive 规则拆成四段：

**Trigger → Condition → Action → Notify**

Trigger 决定什么时候跑；Condition 决定要不要动；Action 是真正执行；Notify 是无论执不执行，都让用户知道发生了什么。

### 1. 触发源

在 OpenClaw 里，触发源可以很轻量地接入：

- `cron`：定时巡检，例如每 10 分钟查一次 CI 状态。
- MCP resource 变化：某个文件、数据库表、API 返回值变化。
- Webhook：GitHub、GitLab、监控系统推送事件。
- 文件系统 watcher：本地目录新增文件、日志出现关键字。

可以先从 cron 开始，因为它最容易观测和控制。

### 2. 条件判断

不要让 Agent 直接“看到事件就执行”。先让它输出一个判断结果，例如：

```json
{
  "should_act": true,
  "reason": "CI 最近 3 次运行均失败",
  "evidence": ["run_id: 1023", "run_id: 1024", "run_id: 1025"]
}
```

这个 JSON 可以写进日志，方便回放和审计。如果以后误报，你能知道当时 Agent 是基于什么证据做的判断。

### 3. 动作白名单

主动执行的 Agent 应使用单独的 MCP tool 白名单。只允许发消息、创建 issue、打 tag、重启无状态服务这类低风险动作。禁止删除、写生产库、修改权限。

示例规则：

```yaml
proactive_rules:
  - name: ci_failure_watch
    trigger:
      type: cron
      schedule: "*/10 * * * *"
    condition:
      mcp: github
      tool: list_failed_runs
      assert: "len(result) > 0"
    action:
      mcp: im
      tool: send_message
      params:
        channel: "#build-alerts"
        template: "CI 失败：{{runs}}"
    cooldown: 30m
    shadow: true
```

这里 `shadow: true` 是关键：前期只记录“将要执行什么”，不真正发送消息。

### 4. 通知与确认

低风险动作可以自动执行，但必须通知。高风险动作建议生成待确认卡片，让人点一下再执行。

## 踩坑点

### 1. 触发风暴

Webhook 重复推送、同一事件被多个 trigger 捕获，都会导致 Agent 反复执行。解决方式是在执行前用 Redis 记录 `event_id` 或 `task_key`，重复则跳过。

### 2. 上下文膨胀

定时任务如果每次都把大量原始状态塞进 Agent 上下文，会迅速消耗 token，还容易导致误判。更好的做法是让 MCP tool 内部做过滤，只返回摘要，例如“最近 3 次失败，平均耗时 4.2 分钟”。

### 3. 误报多，用户静音

proactive 能力最常见的死法，就是推了一堆不痛不痒的信息。用户被打扰几次后，要么静音，要么直接关掉。上线前先跑影子模式一周，看 would-do 日志里有多少是真正值得通知的。

### 4. 权限过大

不要复用日常对话 Agent 的全量 MCP 工具。为 proactive Agent 单独配置 allowlist，只读 + 有限写。

### 5. 循环触发

Action 本身可能产生新事件，又被 Trigger 捕获。比如 Agent 自动创建 issue，而 webhook 又通知“有新 issue”。需要在事件源里标记 `source: proactive_agent`，并让 Condition 忽略自己产生的事件。

### 6. 多实例重复执行

如果 OpenClaw 有多个实例或调度节点，cron 任务可能同时跑。用分布式锁或限制单实例调度。

## 可复用建议

1. **先影子，后真实**：所有规则默认 `shadow: true`，跑一周后人工复核 would-do 日志再开启。
2. **每个主动动作必须带 reason 和 source**：否则出了问题无法追溯。
3. **限制频率**：每条规则设置 `cooldown` 和 `max_per_day`，防止一波突发事件刷屏。
4. **保留 kill switch**：提供一个全局开关，或者 `/proactive off` 指令，能立即停止所有主动执行。
5. **执行结果写回记忆**：避免同一问题反复告警，也能让 Agent 后续判断更准确。
6. **从低风险、高价值场景切入**：日报聚合、过期提醒、状态巡检、证书到期检查，而不是自动修 bug 或自动改配置。

## 总结

Proactive 能力不是让 AI 更“主动”，而是让它在受控边界内替你盯住那些不需要实时决策的事。OpenClaw 的 MCP 工具链和 Agent 调度能力足够落地这类需求，但关键是把 Trigger、Condition、Action、Notify 拆开，配合影子模式、白名单和冷却机制。

一个凌晨三点的误报告警，比没有 proactive 能力更让人崩溃。先用工程手段控制住“主动性”，再谈智能。

---

