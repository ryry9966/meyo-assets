---
title: 让你的 AI 助手学会“主动出击”：基于 MCP 与 Cron 的 Proactive Agent 工程实践
feedId: 32033
source: 综合讨论
publishedAt: 2026-08-08
---

## 背景

绝大多数 AI Agent 仍停留在“问一句、答一句”的被动模式里。你让它检查服务器状态，它就去查；你不开口，它就静静等着。但当我们需要周期性巡检、事件驱动的响应，或者希望 Agent 在问题发生前就给出预警时，这种被动体验就跟不上了。

Proactive（主动式）AI 能力的核心很简单：**不等用户开口，系统自行感知环境变化并执行任务**。对 OpenClaw、Agent 框架、MCP 和自动化实践者来说，这意味着我们要把 Agent 从“对话机器人”改造成“后台协作者”，让它能按时间、事件或状态变化自主工作。

本文将以一个真实可复现的工程方案为例，展示如何给 Agent 添加 proactive 能力，同时避免常见的可靠性陷阱。

## 问题拆解

让 Agent 变得 proactive，本质是在现有 Agent 逻辑外围加一层**触发与调度机制**，再让它通过标准化工具获取外部信号。典型的 proactive 流程需要解决三个问题：

1. **何时触发**：基于 cron 时间表、webhook 事件，还是状态轮询？
2. **如何感知**：通过哪些工具获取环境信息，并判断是否需要行动？
3. **如何安全执行**：如何避免误操作、无限重试、资源耗尽？

我们选一个常见场景：**每日自动检查 GitHub 仓库的未关闭 Issue，如果新增超过阈值，就汇总推送到企业微信**。这个场景涵盖定时触发、条件判断、工具调用和通知，足够通用。

## 做法与步骤

### 1. 准备 MCP 工具：让 Agent 能“看见”

我们使用 MCP （Model Context Protocol） 服务器将外部服务能力暴露成 Agent 可调用的工具。针对上述需求，写一个最小可用的 Python MCP server，提供两个工具：

- `list_recent_issues(owner, repo, since_days)`：返回指定仓库最近 N 天创建的 issue 列表及数量。
- `send_wework_message(content)`：通过 webhook 发送消息到企业微信群。

工具实现使用 `mcp` 库（`pip install mcp`），核心结构如下：

```python
from mcp.server import Server, NotificationOptions
from mcp.server.models import InitializationCapabilities
import httpx

server = Server("github-wework")

@server.tool()
async def list_recent_issues(owner: str, repo: str, since_days: int = 1) -> dict:
    """List recent issues from a GitHub repo."""
    since = (datetime.utcnow() - timedelta(days=since_days)).isoformat() + "Z"
    url = f"https://api.github.com/repos/{owner}/{repo}/issues?since={since}&state=open"
    async with httpx.AsyncClient() as client:
        resp = await client.get(url, headers={"Authorization": f"token {GITHUB_TOKEN}"})
        issues = resp.json()
        return {"count": len(issues), "titles": [i["title"] for i in issues[:5]]}

@server.tool()
async def send_wework_message(content: str) -> dict:
    """Send a markdown message to WeCom."""
    url = "https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=YOUR_KEY"
    async with httpx.AsyncClient() as client:
        resp = await client.post(url, json={"msgtype": "markdown", "markdown": {"content": content}})
        return resp.json()
```

将 MCP server 启动后，在 OpenClaw 或其他 Agent 框架的配置中添加此 MCP 连接。

### 2. 构建主动触发链路：用 cron 驱动 Agent 例行任务

Proactive 的“主动”需要一个外在触发器。这里不改造 Agent 内核，而是通过操作系统的 cron 定时向 Agent 发送一条“检查指令”。这符合“触发器在外部，决策在 Agent”的架构，解耦更好。

例如，每早 9 点执行一次检查，可添加 cron 条目：

```cron
0 9 * * * /usr/local/bin/openclaw run --task "主动检查任务：调用 github-wework MCP 工具的 list_recent_issues，owner=myorg，repo=backend，since_days=1。如果 issue 数量大于 3，就用 send_wework_message 发送汇总消息，格式为：'今日新增 Issue {count} 个：{titles}'。否则不发送。" >> /var/log/openclaw_proactive.log 2>&1
```

这里的关键是：**用自然语言描述 proactive 任务**，让 Agent 自己去理解条件判断和操作。OpenClaw 的 `run` 命令支持一次性任务，执行完毕即退出，避免长进程消耗资源。

### 3. 条件判断与闭环

Agent 收到 cron 任务后，事实上完成了一个 “感知 → 决策 → 执行” 的闭环：

- 感知：通过 MCP 工具拿到实际数据。
- 决策：根据配置的阈值（issue 数大于 3）决定是否通知。
- 执行：调用消息推送工具。
- 日志：把结果写到指定日志文件，方便排障。

这样，一个完全不需要用户干预的主动任务流就搭建完成了。

### 踩坑点

实际落地时，下列问题几乎必踩：

- **频率控制与 API 限流**：把 cron 频率设太高，GitHub API 可能直接 403。建议对每个主动任务设置最小调用间隔（例如 30 分钟），并在 MCP 工具内部做短时缓存。
- **静默失败**：MCP 工具若因网络抖动异常退出，Agent 可能拿不到返回值直接跳过，造成“明明该通知却无声”。必须给工具调用添加重试（2 次为宜），并在 Agent 任务描述中加入“如果工具调用失败，改用 fallback 方式记录错误至文件”。
- **无人类介入的风险**：自动操作一旦涉及写操作（如关闭 issue、重启服务），一定要加 human-in-the-loop 审批。可以在 Agent 执行危险动作前，使用 `request_human_approval` 类 MCP 工具（或通过企微发消息并等待回复）来确认。
- **日志与可观测性缺失**：cron 的 stdout/stderr 仅记录原始输出，难以回溯 Agent 内部决策。建议在 Agent 系统指令中要求“每次运行均用结构化格式输出决策原因和最终操作”，并送入集中日志系统。

### 可复用建议

把 proactive 能力抽象为可复用的工程模式，需要以下几点：

1. **用状态机管理任务生命周期**：为每个主动任务定义一个轻量状态（如 `idle→checking→evaluating→executing→done`），存储在本地文件或 Redis 中。若一次执行中断，下次可恢复或跳过。
2. **解耦触发与执行**：触发层（cron/webhook/消息队列）只负责任务下发的信号，Agent 作为执行器无状态运行。两者通过标准命令或 API 通信，易于扩缩。
3. **对所有写操作做“干跑”和回滚预案**：主动行为的最坏后果是删库跑路。开发阶段先在只读工具上跑通，逐步开放写权限，并确保每次写操作都有对应的回滚能力（如通过 Terraform/版本控制还原）。
4. **将主动任务的配置显式化**：不要将阈值、通知渠道等信息硬编码在 cron 命令里。维护一个 YAML 配置文件，cron 命令引用配置指针，Agent 再通过 MCP 工具读取配置。这样调整阈值时无需改 crontab。

## 总结

Proactive 能力将 AI 助手从“被调用的函数”提升为“主动协作的工程伙伴”。借助 MCP 标准化工具、简单的 cron 调度、以及 Agent 自身的推理决策能力，我们可以用较低的成本搭建出每天自动检查、自动预警、甚至自动修复的工作流。但切记：**主动意味着更大的责任**，工程上需要投入额外心力在限流、容错、人机审批和状态追踪上。控制好边界，你的 Agent 就能安全、可靠地“不等开口就把事办了”。

---

