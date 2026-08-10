---
title: Agent 的 HEARTBEAT.md：让 AI 主动做事而不是等你提问
feedId: 32458
source: 综合讨论
publishedAt: 2026-08-11
---

# Agent 的 HEARTBEAT.md：让 AI 主动做事而不是等你提问

## 一、背景与问题

当前大多数 Agent（无论基于 LangChain、AutoGPT 还是 OpenClaw 生态）仍然以“举手模式”工作：用户通过 chat 或 command 发起一个任务，Agent 执行、返回结果，然后安静等待。在自动化巡检、信息监控、定时报告这些场景里，这种被动模式非常低效——你不得不自己记着去问它，本质上只是把命令行换成了自然语言。

即便一些框架提供了“循环任务”或“计划器”，它们往往要求你用代码或配置文件精确描述触发器与动作，失去了 Agent 对模糊意图的理解优势。我们缺的是一个既能让 Agent 理解*该什么时候干什么事*，又保持轻量、无需复杂调度引擎的工程化约定。

## 二、HEARTBEAT.md 是什么

HEARTBEAT.md 是一份放在项目或 Agent 工作目录下的 Markdown 文件，它像一个“心跳清单”，告诉 Agent 在空闲时应该主动检查并执行哪些任务。文件本身是人类可读的，同时会被一个极简的运行器（runner）定期喂给大模型，让模型自行判断当前时间、状态是否满足任务条件，然后调用工具执行。

思路来源于基础设施中的 health check 和 cron，但用自然语言表达能力代替严格的 crontab 语法。举个例子：

```markdown
# HEARTBEAT.md

## 早晨简报
- 每个工作日 9:00，查询过去 24 小时团队 Git 提交记录与未解决的 issue
- 如果发现超过 3 个未解决的高优 issue，通过 Slack MCP 通知 #team-alert 频道
- 将简报总结后写入 daily-digest.md

## 天气提醒
- 每天 7:00、19:00 查询城市天气
- 如果降雨概率 > 70%，推送一条带雨具提醒的桌面通知

## 数据巡检
- 每 4 小时检查一次 API 端点 /health
- 若响应码非 200 或延迟 > 2s，在日志中记录，并触发 PagerDuty 告警
```

这里没有严格的 cron 表达式，没有代码，但一个配备了合适工具（MCP server 提供文件读写、终端、通知、HTTP 请求等能力）的 Agent 完全能够理解并执行。

## 三、实现步骤（以 OpenClaw + MCP 为例）

下面给出一个能在 30 分钟内跑起来的最小实现，面向 OpenClaw 社区惯用的插件式架构。

### 1. 准备 HEARTBEAT.md
在工作空间根目录创建 `HEARTBEAT.md`，用自然语言描述主动任务。建议为每一项明确：**触发条件**（时间、间隔、事件）、**执行动作**、**失败处理**。这也是工程化落地的关键：你需要像写运维 runbook 一样写它，而不是只扔一句模糊指令。

### 2. 编写 Runner 脚本
Runner 是一个极简的循环或者 cron 触发的 Python/Shell 脚本，核心逻辑只有三步：

1. 读取 `HEARTBEAT.md` 内容
2. 构造一个系统提示词，要求模型“你是一个定时任务调度器，当前时间为 {now}，请阅读下面的 HEARTBEAT.md，判断是否有需要执行的任务。如有，请使用工具执行”。
3. 调用 OpenClaw 的一次性推理（例如 `openclaw task --prompt-file /dev/stdin` 或通过 Python SDK）

以 Python 伪代码为例，避开具体 CLI 绑定，核心逻辑如下：

```python
import datetime, subprocess

with open("HEARTBEAT.md") as f:
    heartbeat = f.read()

prompt = f"""当前时间: {datetime.datetime.now()}
你是任务调度Agent。请阅读 HEARTBEAT.md 并决定是否执行任务。可以使用的工具有: slack_notify, desktop_notify, http_request, write_file。请逐步推理并执行。"""

# 通过管道交给 OpenClaw 执行（假设 openclaw 支持标准输入）
proc = subprocess.run(
    ["openclaw", "--mcp-config", "mcp.json", "--task", "-"],
    input=prompt, capture_output=True, text=True
)
print(proc.stdout)
```

### 3. 配置 MCP 工具
在 `mcp.json` 中声明所需的 MCP 服务器，如文件系统服务器（读写日志）、Slack 服务器、系统通知服务器、HTTP 客户端等。确保工具名称与 HEARTBEAT.md 中提及的动作可以映射。Runner 不需要硬编码任何具体业务，一切由模型根据文档和可用工具自主决策。

### 4. 调度 Runner
在开发机上用 `cron` 或 `systemd timer` 每 1-2 分钟触发一次 runner。虽然模型推理有成本，但这个间隔对于大多数主动检查完全够用，且能避免频繁调用大模型。

## 四、踩坑点（真实排障记录）

1. **自然语言时间解析不可靠**  
   让模型判断“是不是每个工作日9:00”很容易出错，因为模型可能因时区、日期格式产生偏差。解决方法是 Runner 在注入当前时间时，格式要极度显式，例如 `2025-03-17 09:02:00 (周一)`，并在系统提示里强调“只在时间匹配±2分钟内执行”。

2. **Agent 无限循环或重复执行**  
   如果 HEARTBEAT.md 里写了“每天9点做简报”，但 runner 每分钟都在跑，模型可能会每分钟都认为该执行。必须增加*幂等性控制*：让 Agent 在执行前先检查上次执行记录（例如读取 `heartbeat_log.json`），或者强制模型“每个任务在指定时间窗口内只触发一次，非首次忽略”。

3. **工具权限过大导致副作用**  
   主动 Agent 可能因为误解而发送大量消息或错误删除文件。务必限制 MCP 服务器的权限，例如允许通知类工具，但禁止文件删除、禁止执行 shell 命令。在生产环境里，建议用只读文件系统 + 白名单工具。

4. **成本与延迟**  
   每分钟调用 LLM 对 API 费用是持续消耗。可以通过本地小模型（7B-13B）做筛选，只在确认有任务时再调用强模型，或者将调度逻辑外置化：runner 先做简单的时间正则匹配，匹配到的任务才给 Agent 推理。

## 五、可复用的工程建议

- **结构化优于自然语言**：在 HEARTBEAT.md 里用 YAML front matter 或 markdown table 定义 `when`、`every`、`action`，降低解析歧义。例如：
  ```
  | id | when  | every | action | tools |
  |----|-------|-------|--------|-------|
  | 1  | 09:00 | 1d    | 生成简报 | git, slack, file |
  ```
- **为 HEARTBEAT 单独配置一个 Agent profile**：在 OpenClaw 中为调度任务创建一个独立的 agent 定义，设置 temperature=0、严格的系统提示，减少幻觉。
- **日志与回放**：将每次 runner 的输出记录到带时间戳的日志文件，方便回溯 Agent 到底做了什么。可以再用另一个 Agent 定期分析日志。
- **渐进式启用**：先只放一个无害任务（例如每小时写一条时间戳到文件），观察一周稳定性后再加入生产通知类任务。

## 六、总结

HEARTBEAT.md 不是一个软件或框架，而是一种 Agent 行为的轻量约定。它把“定时任务”从代码逻辑中剥离，放到人类可读可改的文档里，既保留了自然语言的灵活性，又用 runner 模板实现了工程化可控。在 OpenClaw 这类强调插件、MCP 工具链的 Agent 环境里，这种模式让 AI 从被动回答者变成主动值守的“数字同事”——你需要做的，只是在 HEARTBEAT.md 里写下它该操心的事。

如果你的团队已经用 Agent 做了不少自动化，不妨尝试为它创建一个 HEARTBEAT.md，看看它会在你休息时偷偷做些什么。

---

