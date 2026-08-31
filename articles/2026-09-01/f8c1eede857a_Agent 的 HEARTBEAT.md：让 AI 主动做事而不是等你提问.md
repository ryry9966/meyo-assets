---
title: Agent 的 HEARTBEAT.md：让 AI 主动做事而不是等你提问
feedId: 35637
source: 综合讨论
publishedAt: 2026-09-01
---

## 背景

大多数 Agent 是“被动响应”的：你输入一句指令，它执行一次。这在对话式助手场景里没问题，但一旦涉及定时巡检、状态监控、待办提醒、日志清理这类重复性工作，被动模式就很别扭——要么你每隔一段时间手动去问一次，要么额外搭一套 cron 胶水脚本，而 Agent 本身对“时间到了该干活”这件事毫无感知。

HEARTBEAT.md 是一个轻量的工程化约定：在工作区根目录放一个 Markdown 文件，里面结构化地描述“这个 Agent 应该主动做哪些事、多久做一次、怎么做”。Agent 每次被唤醒时先读取这个文件，检查有没有到期任务，然后自主执行并回写状态。它不依赖复杂框架，核心只是“一个文件 + 一个定时入口 + 一套工具调用约定”。

## 问题：被动 Agent 的局限

- 不会主动感知时间流逝，用户不输入就不工作。
- 定时任务（cron）只能跑固定脚本，无法利用 Agent 的上下文理解与工具编排能力。
- 自主行为缺少可审计、可版本控制的配置来源，出了事很难追溯“它为什么这么做”。
- 每次靠用户临时口述任务，容易遗忘、不一致，也不利于团队共享。

HEARTBEAT.md 把这些缺口补上：让 Agent 有一个“心跳”信号源，明确地知道“我该在什么时候主动做什么”。

## 做法：一个最小可用的 HEARTBEAT.md 机制

**第 1 步：定义 HEARTBEAT.md 结构**

建议使用 YAML front matter 放在文件开头，后面可以跟一些人类可读的说明。示例：

```yaml
heartbeat:
  interval: 5m
  tasks:
    - id: check-site
      schedule: every 10 minutes
      action: fetch_url
      params: { url: "https://example.com/health" }
      expect: { status: 200 }
      last_run: null
      status: pending
    - id: clean-tmp
      schedule: at 03:00 daily
      action: run_script
      params: { script: "./scripts/cleanup.sh" }
      last_run: null
      status: pending
```

关键字段说明：
- `id`：任务唯一标识，方便日志追溯。
- `schedule`：用自然语言或简单间隔表达，避免复杂 cron 让 Agent 产生理解偏差。
- `action`：指定要调用的工具或 MCP 能力，比如 `fetch_url`、`run_script`、`send_message` 等。
- `params`：该动作的参数。
- `last_run` / `status`：每次心跳后更新，形成闭环。

**第 2 步：加一个定时入口**

可以用外部 cron 或 systemd timer 每 5 分钟调用一次 Agent CLI，传入固定提示词，例如：

```
openclaw run --prompt "读取工作区根目录的 HEARTBEAT.md，执行所有到期任务，更新 last_run 和 status，并输出本次心跳日志。"
```

如果你的 Agent 运行时支持 MCP 的 scheduler/timer 插件，也可以直接由 MCP 服务触发，效果一样。关键是要让 Agent 在固定节奏下被唤醒。

**第 3 步：让 Agent 按契约执行**

Agent 读取 HEARTBEAT.md 后，解析 YAML，筛选出当前时间已到期的任务（`last_run` 为空或距离上次执行超过 schedule），然后逐个调用对应工具执行。执行完成后写回 `last_run` 和 `status`，并生成一条简短的 heartbeat log（可以追加到 `heartbeat.log`，也可以输出到控制台）。

**第 4 步：限制与边界**

在提示词中明确要求 Agent 只允许执行 HEARTBEAT.md 中声明的 action，禁止调用未列出的工具。同时建议将 Agent 运行在受限环境（例如只读文件系统、白名单网络地址、docker 沙箱）。

## 踩坑点

1. **Agent 对 cron 表达式解析不可靠**：有些模型会把 `*/10 * * * *` 当成乱码。建议直接用自然语言描述频率，如 “every 15 minutes” 或 “at 02:00 daily”，让模型自己换算成间隔。
2. **并发重复执行**：如果多个心跳进程同时触发，可能把同一个任务跑两遍。在入口处加一个锁文件（如 `heartbeat.lock`），或依赖外部调度器保证单实例。
3. **权限失控**：让 Agent 主动执行 shell 命令风险很高。只允许白名单 MCP 工具，不要把原始 shell 直接暴露给模型。
4. **日志膨胀**：每次心跳都写一行日志，时间久了文件会很大。建议按天轮转，或只保留最近 200 条。
5. **网络抖动导致误报**：外部服务短时不可用会触发失败状态。需要给任务加简单重试和退避，避免 Agent 反复报警或重复执行危险操作。

## 可复用建议

- 把 HEARTBEAT.md 当成活的“任务队列”，而不是文档；每个任务的执行结果必须回写，保持状态一致。
- 任务设计要幂等：同一个任务跑两次不应产生副作用。例如检查网站只读，清理脚本也要能安全重复运行。
- 心跳间隔别太短，否则 token 消耗和噪声都会上升。日常监控 5–15 分钟一次足够。
- 结合 MCP 提供标准化工具，例如 `fetch_url`、`write_file`、`send_message`，不要让 Agent 自己去拼 shell 命令。
- 定期人工审查 heartbeat log，观察 Agent 自主行为是否符合预期，必要时收紧任务范围。

## 总结

HEARTBEAT.md 的价值在于：用极低的成本，把被动 Agent 改造成一个按节奏主动干活的执行者。它不依赖复杂框架，只靠一个约定文件、一个定时入口和一套工具调用。适合监控巡检、定时报告、简单运维清理等场景。只要控制好权限和任务边界，它就能稳定地替你盯着那些“不需要每次问，但需要定期做”的事。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/7f31687c74246ffb.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/af2100d7e180cf45.png)

