---
title: Agent 的 HEARTBEAT.md：把被动问答改成主动巡检
feedId: 35304
source: 综合讨论
publishedAt: 2026-08-30
---

## 背景

多数 Agent 工作流是“提问-回答”：你发消息，它调用 MCP 工具，给结果。人不在时，它不会自己看仓库、日志或 TODO。结果就是自动化只在会话里存在，离开键盘就停。

OpenClaw/MCP 的好处是工具已经标准化，但缺一个“什么时候该看一眼”的调度层。HEARTBEAT.md 是我在项目里放的一个小文件，给 Agent 定义心跳：每 N 分钟读一次，按规则判断有没有值得报告或处理的变化。

它不替代 workflow，也不让 Agent 自由发挥。它更像值班表：什么要看、什么能改、什么时候闭嘴。

## 问题边界

主动 Agent 容易做成两个极端：要么只回“一切正常”刷屏，要么看到什么都想 commit/deploy。真正要解决的是：

- 变化很多，但只有少数需要上报；
- 动作有副作用，必须白名单；
- 同一件事不能每次心跳都报；
- 没有变化时保持安静。

所以 HEARTBEAT.md 不是 prompt 大全，而是触发器、动作边界和去重规则的声明。

## 做法

我把它放在项目根目录 `.openclaw/HEARTBEAT.md`，和一个 `heartbeat-state.json` 去重状态文件放在一起。

一个最小文件长这样：

```markdown
---
cadence: 30m
scope:
  - git status
  - logs/errors
  - TODO.md
allow_actions:
  - append_todo
  - create_draft_issue
  - send_summary
require_approval:
  - git_commit
  - run_tests
dedupe:
  state: .openclaw/heartbeat-state.json
  key: "{scope}:{fingerprint}"
rules:
  - if untracked files > 3 and no draft note: append TODO
  - if repeated error in logs >= 5: create_draft_issue
  - if TODO.md stale > 3 days: send_summary
stop_if:
  - no changes
  - already_notified_today
```

然后接一个定时任务。我不用复杂调度，cron 就够：

```bash
*/15 * * * * cd /path/to/project && your-agent-cli heartbeat \
  --rules .openclaw/HEARTBEAT.md \
  --state .openclaw/heartbeat-state.json \
  --log .openclaw/heartbeat.log
```

实际命令取决于你的 Agent CLI 封装；关键是每次只做三件事：读规则、读状态、按 `allow_actions` 输出。前两周把 `send_summary` 以外的动作都关掉，只观察。

## 踩坑点

1. **去重没做好**。第一次跑会报一堆“新变化”，第二次又报一遍。必须给每个信号算指纹，写进 state；同一个 fingerprint 不重复报。
2. **动作权限过大**。如果 `allow_actions` 里混进 `git_commit` 或 `deploy`，Agent 会在凌晨替你做决定。默认只读，写操作只允许 append/draft。
3. **频率和 token 成本**。30 分钟一次跑完整上下文不便宜。可以先让便宜模型做 triage，过滤到真正异常再进大模型。
4. **cron 环境不一致**。PATH 里没有 MCP 工具或密钥，定时任务失败时 Agent 不知道。日志要落到文件，失败也记。
5. **HEARTBEAT.md 变成垃圾桶**。规则越加越多，最后每次心跳都慢且吵。建议文件不超过 80 行，规则不超过 8 条。

## 可复用建议

- 第一周只跑 `report-only`，输出到日志或群，不做任何写操作。
- 用 `dedupe` + `stop_if` 控制安静度，而不是让 Agent 自己判断“要不要说”。
- 每个动作必须能在 state 里追溯到上一次触发时间。
- HEARTBEAT.md 纳入版本管理，改动要 review。它决定 Agent 的边界，比普通 prompt 更该审。
- 把心跳当探针，不要当自主员工。真正有副作用的事，仍应回到人工确认。

## 总结

HEARTBEAT.md 的价值不是让 Agent 更聪明，而是给它一个可审计的“主动层”。在 OpenClaw/MCP 环境里，它把分散的工具调用变成有节律的巡检，把“你没问所以没看”变成“按规则看了，按边界做了，按去重保持安静”。

如果你已经有定时任务和 MCP 工具，可以先从一个只读的 `HEARTBEAT.md` 开始，开 `report-only`，跑一周再决定放哪个低风险写动作进去。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/698550dda26da39d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/b34c853193a06ac2.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/a298d8dc7b4b7efe.png)

