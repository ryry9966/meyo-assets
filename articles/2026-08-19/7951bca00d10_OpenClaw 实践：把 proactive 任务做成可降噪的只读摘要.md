---
title: OpenClaw 实践：把 proactive 任务做成可降噪的只读摘要
feedId: 33849
source: 综合讨论
publishedAt: 2026-08-19
---

## 背景

大部分 agent 默认是“回复式”的：你问一句，它动一下。真正省心的场景往往相反——周一早上九点，它已经汇总好本周要处理的安全告警；CI 失败率异常时，它先在内部跑一遍只读检查，再决定要不要通知你。

OpenClaw 具备 trigger 能力，可以把这类事从“等人开口”变成“条件触发”。但 proactive 不等于每天定时发鸡汤。它更像一个需要控制半径的巡检进程：能主动发现，但不该主动打扰。

## 问题

直接给 proactive 任务接通知，通常会踩到几个坑：

- 时区错乱，cron 默认 UTC，国内用户经常在半夜收到“早安摘要”。
- 重复触发，cron 和 webhook 同时监听一个事件，同一条告警推三遍。
- 上下文膨胀，每次触发都复用同一 session，历史消息越积越多，总结变慢且被污染。
- 权限过大，把全量 API token 塞给 proactive 任务，一旦误判就是实际操作。
- 没有静默机制，开会、休假时照样推送，最后被用户整体静音。

## 做法/步骤

### 1. 先做只读摘要，不做执行

第一版 proactive 任务只允许读取数据、生成摘要、发送通知。不要一上来就让它“自动合并 PR”“自动重启服务”。以安全巡检为例：每个工作日上午 8:30，汇总 GitHub 未关闭的 critical issue 和 Dependabot 告警。

OpenClaw 中可配置类似 trigger：

```yaml
triggers:
  - name: security-digest
    schedule: "30 20 * * 1-5" # UTC，国内 8:30
    prompt: |
      Run security digest:
      1) Use GitHub MCP to list open critical issues.
      2) Use Dependabot MCP to list active alerts.
      3) Summarize in Chinese, under 200 words.
      4) If nothing, reply exactly "NOTHING".
    session: proactive
    timezone: Asia/Shanghai
```

这里的关键不是 prompt 写得多聪明，而是**返回哨兵值**。空结果时让 agent 输出 `NOTHING`，外层判断不再推送，从源头降噪。

### 2. 给 proactive 任务独立 session 和工具白名单

不要让它混在主对话里。单独 session 可以避免上下文污染，也方便审计。工具侧只给 GitHub MCP 的读取权限，禁止 write、merge、delete。OpenClaw 支持按 trigger 限制工具时，优先开启；不支持时，至少使用只读 token。

### 3. 增加静默期和去重

在状态存储里维护一个 `silence_until` 字段。用户发 `/quiet 4h` 时写入时间戳，cron 触发后先检查是否在静默期内。去重可以用自然键：例如日期 + 仓库名 + issue ID，推过的组合不再推。

### 4. 从 dry-run 到 notify，再到 execute

每个 proactive 任务都设计三个阶段：

- `dry-run`：只记录要做什么，不发通知。
- `notify`：发送摘要，不做执行。
- `execute`：执行具体动作，但必须有确认或回滚路径。

前两个阶段足够覆盖大部分需求。真正需要 execute 的，建议再拆出一个单独任务，并加二次确认。

## 踩坑点

- **时区**：cron 默认 UTC，必须在配置里显式设置 `timezone`，否则所有时间判断都会错位。
- **重复触发**：cron 和 webhook 同时存在时，同一事件会产生多条通知。加幂等键，比如事件 ID 或日期。
- **上下文膨胀**：每次触发都应使用新 session 或显式清理上下文。长 session 会让 agent 把旧摘要当成新信息。
- **误判执行**：proactive 任务里的“主动”很容易被误写为“自动操作”。权限限制在只读层，是成本最低的安全网。
- **不可观测**：如果一条主动任务触发了却没通知，你很难发现问题。每条推送都带上 trigger 名、时间、耗时、去重 ID，方便回溯。

## 可复用建议

- **一个 trigger 只做一件事**。便于暂停、排障和判断责任边界。
- **优先用 MCP 工具，而不是裸 shell 或 HTTP 请求**。MCP 返回结构化数据，权限更好控制。
- **维护 proactive 任务清单**：记录 trigger 名称、负责人、静默方式、回滚方式、当前状态。
- **推送内容要可解释**：不要只发一句“有异常”，而要带上“哪个 trigger、哪个数据源、触发条件是什么”。
- **把 proactive 当巡检，不当秘书**。先让它学会安静地发现，再逐步让它开口说话。

## 总结

proactive 能力不是让 AI 更“主动”，而是让它在合适的边界内降低人的反应成本。工程上最稳妥的做法，是先把 proactive 任务做成**只读摘要 + 降噪控制 + 幂等去重**，跑稳之后再逐步放宽执行权限。它会主动开口，但你知道它什么时候该闭嘴。

---

