---
title: 让 AI 助手“不等你开口”：用 OpenClaw + Cron + MCP 搭建 Proactive Agent
feedId: 31863
source: 综合讨论
publishedAt: 2026-08-06
---

## 一、背景：对话式 Agent 的局限

默认情况下，基于 LLM 的 Agent（包括 OpenClaw）工作流都是被动响应的：用户输入一句 prompt，Agent 开始推理、调工具、输出结果。这适合交互式场景，但**运维、监控、周期性报告**等自动化任务，恰恰要求 Agent 能在没有人开口的情况下主动干活。

所谓 proactive（主动）能力，是指 Agent 能够按时间计划或外部事件触发，自主执行一系列工具调用，完成信息采集、判断、通知等闭环操作，而不需要人类守在对话框前。落到工程实践里，就是我们常说的“定时巡检”“条件告警”“自动日报”这类需求。

目前 OpenClaw 尚缺少原生调度引擎；但它提供了足够灵活的 CLI/API，并且可以通过 MCP 接入天气、数据库、Slack、邮件等外部工具。这为我们在外部搭建一个轻量调度层创造了条件。

## 二、问题拆解

要把一个被动 Agent 改造成主动干活，需要解决三个核心问题：

1. **触发机制** —— 谁在什么时候叫醒 Agent？  
2. **上下文注入** —— 每次醒来时 Agent 需要知道该干什么、有哪些工具可用、输出到哪里。  
3. **错误与状态管理** —— 任务执行可能失败，某些场景还需要跨执行周期记住状态（例如“上次检查到的错误计数”）。

对应技术选择：用系统 cron 或 Python 的 `schedule` 库负责触发；用一个固定的系统级 prompt 描述 Agent 的角色、可用工具以及每次执行的标准流程；通过外部文件或轻量数据库保存少量持久化状态。

## 三、实现步骤

### 1. 准备 OpenClaw 环境和 MCP 工具

假设你已有 OpenClaw 的基本运行环境，并配置好至少两个 MCP 服务器：

- **weather** —— 通过某个天气 API 获取实时降水概率；
- **slack** 或 **wecom** —— 发送消息到指定群聊。

在 OpenClaw 的配置文件里确保这些工具在 agent 创建时被加载。

### 2. 编写 Agent 的系统提示

为 proactive 任务专门创建一个 agent 定义。系统提示示例如下：

```
你是一个主动运维助手。当收到指令 "DAILY_CHECK" 时：
1. 调用 weather 工具获取今天北京市的降水概率；
2. 如果降水概率 > 60%，调用 slack 工具向 #ops-alerts 频道发送消息：“今日有雨，请检查户外设备”；
3. 无论是否发送消息，都输出简短的操作日志，包含时间和降水概率。
禁止执行任何与上述无关的操作。遇到工具调用失败，重试一次，仍失败则在日志中记入错误。
```

> **踩坑提醒**：这条提示必须极度具体，不给 Agent 留自由发挥空间。曾出现过 Agent 在雨天主动去查消防规范，消耗大量 token 且没有发出告警。工程实践里，“过度智能”反而坏事。

### 3. 编写调度脚本

用一个简单的 shell 脚本 `proactive_check.sh` 封装一次主动任务调用：

```bash
#!/bin/bash
export OPENCLAW_API_KEY="sk-xxx"
LOG_FILE="/var/log/proactive_agent.log"

echo "$(date '+%Y-%m-%d %H:%M:%S') - Triggering daily check..." >> $LOG_FILE

openclaw agent run \
  --agent-id proactive-ops \
  --prompt "DAILY_CHECK" \
  --timeout 30 \
  >> $LOG_FILE 2>&1

if [ $? -ne 0 ]; then
  echo "$(date) - Execution failed" >> $LOG_FILE
fi
```

脚本里关键点：  
- 使用 `openclaw agent run` 而不是启动交互式会话，任务结束后进程自动退出，不占用资源；  
- 通过环境变量注入 API key（不要硬编码）；  
- 对标准输出和错误都做重定向，保留日志。

### 4. 挂载到 Cron

编辑 crontab：

```bash
0 8 * * * /path/to/proactive_check.sh
```

每天早上 8 点触发。如果需要更细粒度（比如每 10 分钟检查一次 API 指标），调整 cron 表达式即可。

### 5. 处理跨周期状态（可选）

如果任务需要记住上一次的结果（例如“连续三次错误才告警”），可以在脚本中通过一个小的 JSON 文件保存状态，并在 prompt 里传入：

```bash
STATE_FILE="/tmp/proactive_state.json"
prev_errors=$(jq '.consecutive_errors' $STATE_FILE)

openclaw agent run \
  --agent-id proactive-ops \
  --prompt "DAILY_CHECK. 当前连续错误次数: $prev_errors" \
  ...
```

Agent 系统提示可以进一步描述如何根据这个计数决定是否告警，以及告警后将计数清零。

## 四、踩坑记录

1. **上下文遗忘** —— 每次调用 `agent run` 都是一次性、无状态的。如果你期望 Agent 记住某些信息，只能通过外部状态文件、数据库或 prompt 注入实现。不要假定它能回想起上一次执行的内容。

2. **工具调用超时与重试** —— MCP 工具可能因为网络问题失败。建议在系统提示里指定简单的重试策略（如“重试一次，立即失败则记录”），并在调度脚本设置较短的超时时间（如 30 秒），避免任务卡死导致 cron 堆积。

3. **环境变量与路径** —— Cron 运行时的环境非常精简，PATH 和虚拟环境往往失效。所有依赖必须在脚本里显式 export 或使用绝对路径。建议先在 shell 里手动 `bash -l` 跑通再放入 cron。

4. **安全边界** —— Proactive Agent 只要有工具调用能力，就必须严格限定可执行的操作。**绝对不要给这类 Agent 开放执行 shell 命令或修改生产数据的权限**，避免误操作。遵循最小权限原则，单独建一套只读或只发消息的工具集。

## 五、可复用建议

- **任务模板化**：将每个主动任务抽象成一个 JSON 配置，包含 agent-id、prompt、cron 表达式、超时时间和状态文件路径，然后由一个统一的 runner 脚本读取并执行，降低运维成本。
- **统一告警出口**：所有主动任务的输出尽量走同一个通知通道（如一个 Slack Incoming Webhook），方便聚合和静默。
- **日志与监控**：给 proactive 脚本加上 `set -euo pipefail`，并把日志接入现有监控系统。如果主动任务连续失败超过 N 次，应触发另一条硬告警。
- **测试先行**：先用 `--dry-run` 或手动指定 prompt 看 Agent 输出是否符合预期，再上 cron。这一步会节省大量排障时间。

## 六、总结

AI 助手的 proactive 能力并不是模型厂商的宣传噱头，而是一个实实在在的工程拼接活儿。核心是用好 Agent 框架的工具调用，再叠加最简单的 Unix 调度机制，就能把原本“等人来问”的对话框变成勤快的自动化巡检员。

OpenClaw 的职责是推理 + 工具编排，Cron 负责时钟，MCP 提供手和眼。三者各司其职，边界清晰，系统就可控、可调试、可复现。

---

