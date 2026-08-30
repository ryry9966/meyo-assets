---
title: 给 Agent 加一份 HEARTBEAT.md：从被动问答到主动值守
feedId: 35446
source: 综合讨论
publishedAt: 2026-08-31
---

## 背景

很多 OpenClaw 用户的 agent 已经接了 MCP 工具、插件和 system prompt，但实际行为仍然是“你问一句，它动一下”。想让 agent 主动做巡检、备份检查、环境感知，光加 cron 不够：如果定时任务只是把一个模糊指令丢给模型，每次执行效果不稳定，还容易产生噪音。

## 问题

定时触发后，agent 最常见的问题是缺少统一任务清单和退出条件。它可能每次都把规则重新理解一遍；可能忘记上次跑过什么；可能把 read-only 检查和有副作用的操作混在一起；也可能一有风吹草动就通知人，最后变成消息疲劳。

## 做法

我把 HEARTBEAT.md 放在工作区根目录，作为 agent 的主动任务“签到卡”。OpenClaw 的 scheduler 每 15 分钟触发一个很窄的 prompt：

```text
Read HEARTBEAT.md. Execute only due items. Respect action_level. Write result to heartbeat-state.json.
```

状态文件单独维护 `last_run`、`next_due`、`status`、`result`，避免重复执行。不要把全部规则塞进 system prompt，否则上下文会越来越重。

一个简单示例：

```md
# HEARTBEAT.md

## check: repo_clean
interval: hourly
condition: dirty_worktree_hours > 12
action_level: read_only
action: list dirty repos
notify: false

## check: local_backup
interval: daily
condition: last_backup_hours > 26
action_level: reversible
action: run backup
notify: false

## check: cert_expiry
interval: daily
condition: days < 14
action_level: notify_only
action: do not renew
notify: true
```

动作分三级：

- `read_only`：自动执行，只读，不通知。
- `reversible`：执行并记录，失败可回滚。
- `notify_only`：只生成待办或通知，不直接操作。

优先通过 MCP 工具执行动作，例如 `git_status`、`backup_create`、`cert_check`，而不是让模型自己临时拼 shell 命令。这样更容易审计，也方便插件复用。

## 踩坑点

1. **不要一上来放高危动作。** 自动重启、自动提交、自动删文件、自动续期证书这些，第一周都不要放进 heartbeat。先跑 read-only 和 reversible。
2. **避免重复执行。** 用状态文件记录 `last_run`，并用锁文件或原子写入防止多个 cron 同时跑同一份 heartbeat。
3. **prompt 要非常窄。** 不要只写“检查系统是否正常”，要给出明确的 condition、action_level 和退出标准。
4. **通知疲劳。** 低优先级不要推送，记录到状态文件就好。只有需要人处理时才通知。
5. **小心状态文件污染。** heartbeat-state.json 要结构化、可追加，不要把所有日志混在一起，否则 agent 下次读取会变慢。

## 可复用建议

- HEARTBEAT.md 不要超过 5 个检查项，先小范围验证。
- 首周强制 dry-run：先输出“将要执行什么”，第二轮确认无误后再真正执行。
- 每个检查项必须有明确的 `condition` 和 `action_level`，没有这两个字段就不允许自动跑。
- 把 heartbeat 当成“环境感知节拍”，而不是另一个待办清单。它应该让 agent 在不同时间点自动发现变化，而不是把更多任务堆给同一个 session。
- 可以配合插件把 heartbeat 动作注册成 OpenClaw 可调用的命令，减少每次的推理成本。

## 总结

HEARTBEAT.md 的价值不是让 agent 更忙，而是让它有一致、有边界的主动节奏。它把“什么时候检查、检查什么、做到哪一步、如何留痕”固化下来，让 agent 从被动问答变成可持续值守。先跑 read-only，再逐步放开 reversible，最后才考虑需要人确认的动作，这是比较稳妥的工程化路径。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/0fc929e1c0a4a49a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/953ca9a406c63caf.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/4e9653d3e386aedc.png)

