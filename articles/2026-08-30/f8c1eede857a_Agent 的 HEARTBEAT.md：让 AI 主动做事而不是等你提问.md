---
title: Agent 的 HEARTBEAT.md：让 AI 主动做事而不是等你提问
feedId: 35299
source: 综合讨论
publishedAt: 2026-08-30
---

## 背景

很多 Agent 实际还是“应答机”：你不发消息，它不动。问题通常不在模型能力，而在于没有给主动行为一个稳定的运行入口。MCP 能接入大量工具，但工具不会自己触发；插件能扩展能力，但缺少固定节奏。于是“请每天帮我检查服务状态”很容易被写进 system prompt，然后在长会话里被稀释、时灵时不灵。

## 问题

把“主动性”交给 system prompt，至少有几个缺陷：

- 模型只在用户输入后运行，长时间不打开会话就没有执行机会；
- 没有可观察状态，不知道跑了没有、为什么没跑、跑了什么；
- 动作边界模糊，容易从“通知我”滑向“自动修复”；
- 多次执行容易重复、抖动作、污染上下文。

所以我更倾向于把主动行为拆成一个可读、可审计的心跳文件，而不是一段自然语言指令。

## 核心做法：HEARTBEAT.md

`HEARTBEAT.md` 不是一个长 prompt，而是一个运行清单。Agent 每个心跳周期读取一次，按“观察—决策—动作”执行。它应该小、明确、可版本化。

目录结构示例：

```text
~/.openclaw/agents/home-ops/
├── AGENT.md
├── HEARTBEAT.md
├── state/
│   ├── last_run.json
│   └── dedupe.json
└── logs/
    └── heartbeat.jsonl
```

`HEARTBEAT.md` 可以写成这样：

```markdown
cadence: 15m
timezone: Asia/Shanghai
dry_run: false

## Observe
- 读 ./inbox/*.md，提取标题和紧急标记
- 读 ./state/last_run.json
- 调用 MCP 工具 service_health

## Decide
- 若 inbox 出现 urgent，且过去 24h 未通知过同一 hash -> Act
- 若 service_health 连续 3 次非 200 -> Act
- 否则只记录 heartbeat，不动作

## Act
allow:
  - send_dm_to_owner
  - create_draft_issue
ask_first:
  - restart_service
  - merge_code
  - delete_files

## Log
- 每轮追加 logs/heartbeat.jsonl：timestamp, trigger, decision, action_id, result
```

触发入口可以是 cron、systemd timer，或 OpenClaw/Agent 宿主提供的 idle hook。通常只需要一个薄命令：

```text
*/15 * * * * openclaw agent heartbeat --agent home-ops --dry-run=false
```

## 落地步骤

1. **先跑只读版**：`dry_run: true`，连续跑 20 轮，只记录 would-do，不执行任何动作。
2. **检查决策质量**：看是否存在误报、重复、越界动作。
3. **只放开一个动作**：例如先允许 `send_dm_to_owner`，其他都放 `ask_first`。
4. **把状态外置**：`last_run.json` 和 `dedupe.json` 不要放进模型上下文，只在需要时读取摘要。
5. **加锁**：同一时间只允许一个心跳实例运行，可以用 lockfile 或原子写入。
6. **每轮留痕**：JSONL 记录时间戳、触发条件、决策结果、动作 ID、Agent 版本，便于回滚。

## 踩坑点

- **心跳噪音**：每 15 分钟跑一次，如果每轮都通知，人会很快忽略。建议“无异常则静默”，只记录日志。
- **权限滑移**：Act 段不要写自然语言，动作必须来自枚举白名单。`ask_first` 里的东西永远不能自动执行。
- **上下文膨胀**：不要让 Agent 每轮读完整日志或大文件。只读摘要、计数、哈希和最近时间戳。
- **重复执行**：用文件哈希、issue ID 或任务 ID 做去重键，不要靠模型判断“这件事我做过没有”。
- **误报抖动**：服务偶尔超时一次不要立即通知。用连续 N 次失败或持续 T 分钟作为阈值。
- **注入风险**：如果心跳会读取 `inbox` 或外部文件，里面的内容可能携带恶意指令。Act 阶段必须限制为预定义动作，不能根据文本动态生成 shell 命令或工具调用。

## 可复用建议

- 心跳间隔默认 15 分钟；除非是实时队列，否则不要短于 5 分钟。
- 把 `HEARTBEAT.md` 控制在 100 行以内，太大说明混入了业务逻辑。
- 观察、决策、动作三段不要混在一起，尤其不要让观察阶段直接触发动作。
- 每次修改 `HEARTBEAT.md` 都要像改代码一样 review，并且记录版本。
- 不要用 HEARTBEAT 替代权限系统。主动行为必须仍然服从工具权限、工作区和审计约束。

## 总结

HEARTBEAT.md 解决的不是“让模型更像人”，而是把主动行为变成一个可观察、可回滚、可关停的轮询任务。它把“主动性”从 prompt 里的模糊期待，变成了一个工程化的小循环：先观察、再决策、最后只做白名单里的动作。

在 OpenClaw/Agent/MCP 场景里，这比单纯换一个更长的 system prompt 更稳定，也更容易复现。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/a74761c77088d78d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/e06c23be58ff5ec9.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/f1bc16260ccbb464.png)

