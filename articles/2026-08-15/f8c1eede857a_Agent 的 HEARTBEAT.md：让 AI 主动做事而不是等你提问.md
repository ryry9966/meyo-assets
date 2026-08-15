---
title: Agent 的 HEARTBEAT.md：让 AI 主动做事而不是等你提问
feedId: 33230
source: 综合讨论
publishedAt: 2026-08-15
---

# Agent 的 HEARTBEAT.md：让 AI 主动做事而不是等你提问

在 OpenClaw 生态里，大多数 Agent 是响应式的：你问一句，它答一句；你给一个任务，它执行一次。这在交互式场景下没问题，但如果你想让它每天检查一次依赖、每周汇总一次日志、或者在某个文件变更后主动跑一遍测试，纯被动模式就很不顺手。

常见的替代方案是写 cron job、接 CI、或者把这些规则塞进 system prompt。但 cron 和 Agent 是割裂的，system prompt 又是静态的，改一次要动配置、重启会话，而且不透明。后来我在几个自动化项目里试了一个更轻的做法：在项目根目录放一个 `HEARTBEAT.md`，让 Agent 每次启动或定时唤醒时先读它，再决定要不要主动做点什么。

## 问题：被动 Agent 的边界

被动 Agent 主要有三个问题：

1. **任务遗漏**：用户忘了问，Agent 就不会做。
2. **上下文漂移**：长期项目里，Agent 不记得自己上次做到哪、哪些任务已经跑过。
3. **不可观测**：主动任务的触发条件、执行历史、权限边界都散落在不同地方，团队很难 review。

HEARTBEAT.md 的思路很简单：把“什么时候该做什么”从隐式的提示词里拿出来，变成一个项目内可见、可版本化、可被 Agent 扫描的约定文件。它不替代调度系统，而是给 Agent 一个轻量的自治入口。

## 做法：一个可扫描的心跳文件

### 1. 定义 HEARTBEAT.md 结构

文件放在项目根目录，内容尽量机器可读，避免大段自然语言。一个最小可用的结构如下：

```markdown
# HEARTBEAT.md
project: repo-analyzer
version: 1

triggers:
  - id: weekly-report
    type: cron
    expr: "0 9 * * 1"
    timezone: Asia/Shanghai
  - id: docs-changed
    type: file_event
    path: "docs/**"
    action: modified

tasks:
  - id: weekly-report
    trigger: weekly-report
    action: run_script
    command: "make weekly-report"
    guardrails:
      allow_network: false
      max_duration_sec: 300
    status: pending
    last_run: null

constraints:
  allowed_commands: ["make", "python scripts/", "git status"]
  forbidden: ["rm -rf", "git push --force"]
  log_file: "HEARTBEAT.log"
```

触发条件和任务分开列，任务通过 `trigger` 字段关联。状态字段 `status` 和 `last_run` 是核心，用来保证幂等。

### 2. 让 Agent 主动读它

在 OpenClaw 里，最简单的方式是在项目级 agent instruction 中加一条硬性要求：

> 每次会话开始或定时唤醒时，先读取项目根目录的 HEARTBEAT.md。如果有满足触发条件且状态为 pending 的任务，按 guardrails 执行，并将结果写回 HEARTBEAT.md 和 HEARTBEAT.log。

更工程化的做法是封装成一个 MCP 工具。比如实现 `get_due_tasks`、`claim_task`、`complete_task` 三个方法，让 Agent 通过 MCP 调用而不是直接改文件。这样可以避免乱写状态，也能在工具层做权限校验。

### 3. 执行循环

Agent 的执行流程可以固定为：

1. 读取 HEARTBEAT.md。
2. 解析 triggers，匹配当前时间和事件。
3. 筛选出 `status: pending` 且满足触发条件的任务。
4. 认领任务：把状态改为 `running`，记录 `started_at`。
5. 在 guardrails 范围内执行动作。
6. 回写状态为 `done` 或 `failed`，记录 `last_run`。
7. 输出一行摘要到 HEARTBEAT.log，方便后续 review。

## 踩坑点

**重复执行**  
如果没有状态回写，Agent 每次唤醒都会重新跑一遍任务。务必在设计上让任务幂等：执行前先检查 `last_run`，或者把任务设计成可重复执行也无副作用的形式。状态字段要由 Agent 自己维护，不能只靠人肉改。

**权限过大**  
HEARTBEAT.md 里写了可执行命令，Agent 可能因为它“觉得应该做”就越界。一定要在 `constraints` 里列白名单，并且在 MCP 工具层做二次校验。不要允许 Agent 自行扩展命令范围。

**触发条件解析**  
尽量避免“每周一早上看看有没有需要更新的东西”这种纯自然语言触发。用 cron 表达式或明确的 file_event 匹配，否则 Agent 容易理解偏差。时间要带时区，文件和路径要能 glob 匹配。

**文件膨胀**  
HEARTBEAT.md 如果写得太长，每次读取会消耗大量 token，影响 Agent 的注意力。建议把大段脚本或详细步骤外置到脚本文件，HEARTBEAT.md 只保留触发条件、任务 ID 和命令入口。

**并发冲突**  
多个 Agent 同时读写 HEARTBEAT.md 会互相覆盖状态。如果可能并发，用锁文件或让所有状态变更走统一的 MCP 服务，避免直接改文件。

## 可复用建议

- **任务原子化**：一个 HEARTBEAT 任务只做一件事，失败时容易定位。
- **dry-run 优先**：加一个 `dry_run: true` 的全局开关，先让 Agent 模拟执行，确认无误后再放开。
- **状态外置**：HEARTBEAT.md 只保留任务定义，运行状态可以放到 `HEARTBEAT.state.json`，避免频繁修改 markdown 导致 git diff 混乱。
- **与现有调度共存**：如果是高频、强时效的任务，仍建议用 cron 或 CI；HEARTBEAT.md 适合低频、需要 Agent 判断上下文的任务。
- **定期人工 review**：每周扫一眼 HEARTBEAT.log，确认 Agent 没有悄悄做奇怪的事。

## 总结

HEARTBEAT.md 不是银弹，它只是一个轻量的自治协议：把触发条件、任务状态、权限边界放进一个版本化文件，让 Agent 有据可依地主动行动。它的价值在于**可见性**和**可回滚**——团队可以 review Agent 为什么做了一件事，也可以随时改文件调整行为，而不需要进 system prompt 里翻找。

如果你已经在 OpenClaw 里挂了不少 MCP 工具，但总觉得 Agent 还是“戳一下动一下”，可以试试在项目里放一个 HEARTBEAT.md。先把触发条件收敛到最小集合，跑通一个闭环，再逐步扩大。

---

