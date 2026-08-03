---
title: 给 OpenClaw 装上“条件反射”：用 Cron + MCP 实现 AI 助手的主动巡检与自愈
feedId: 31430
source: 综合讨论
publishedAt: 2026-08-03
---

## 背景：被动问答的边界

我们用 OpenClaw 接入了各种 MCP 工具、知识库和自动化插件之后，能做的事情已经远超普通聊天机器人。但一个尴尬的现实是：无论能力多强，绝大多数助手仍然是“被动响应”的——你不开口，它不动。

这种模式在“查一下”、“帮我总结”这类按需任务上没问题，但在运维巡检、系统健康监控、定时报告这类需要**主动感知、主动决策**的场景里，就会暴露出明显的短板。你想让 AI 在 CPU 飙高时自动分析进程、在凌晨数据库备份失败时主动告警、在每日开盘前推送关键数据汇总，光靠“等用户来问”是不可能的。

这就引出一个工程上必须解决的问题：**如何以最小侵入成本，给 OpenClaw 装上“不开口就把事办了”的 proactive 能力**。

## 问题拆解

要让助手主动做事，需要解决三个核心节点：

1. **触发时机**：什么时间、什么条件下触发一次决策循环？
2. **感知通道**：助手如何获取当前的真实环境状态（而不是靠用户描述）？
3. **行动闭环**：输出决策结果后，由谁执行动作（发通知、调接口、写日志）？

而 OpenClaw 本身并不内置 cron 或事件驱动的主动调度层，它的能力边界在“接收消息→调用工具/模型→返回回复”这个对话环路里。所以，我们要做的是用一个**薄调度层**把“时间/事件触发”转换成一次对 OpenClaw 的请求，然后借助已有的 MCP 工具获取环境数据，最后把助手回复里的结构化决策捞出来执行。

## 做法：Cron → OpenClaw Agent → MCP 传感器 → 动作

下面以一个真实可复现的服务器健康巡检为例，给出整体架构和关键步骤。最终效果是：每 5 分钟自动检查 CPU 负载和内存使用率，超出阈值时通过企业微信 Webhook 告警，并返回助手给出的进程分析建议。

### 1. 用 MCP 暴露系统指标

写一个最小化的 MCP server，提供 `get_system_health` 工具。这里用 Python 的 `fastmcp` 快速搭建：

```python
# system_sensor_mcp.py
import psutil
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("SystemSensor")

@mcp.tool()
def get_system_health() -> dict:
    return {
        "cpu_percent": psutil.cpu_percent(interval=1),
        "memory_percent": psutil.virtual_memory().percent,
        "top_processes": [
            {"name": p.name(), "cpu": p.cpu_percent(), "memory": p.memory_percent()}
            for p in psutil.process_iter(['name', 'cpu_percent', 'memory_percent'])
            if p.cpu_percent() > 0
        ][:5]
    }
```

启动 MCP server 时要暴露 stdio 或 SSE 接口供 OpenClaw 连接。

### 2. 在 OpenClaw 中配置“巡检 Agent”

在 OpenClaw 的 agent 配置里，给这个巡检场景单独定义一个 agent（或 skill），系统提示词大致如下：

```
你是一个服务器巡检助手。当被调用时，会用 get_system_health 获取实时指标。
如果 CPU > 80% 或内存 > 85%，输出一个严格 JSON：
{
  "action": "alert",
  "severity": "warning|critical",
  "message": "简洁中文告警内容",
  "suggestion": "基于进程信息给出的分析建议"
}
如果一切正常，输出：
{
  "action": "ok"
}
只输出 JSON，不要带额外解释。
```

将刚才的 MCP server 作为工具源挂到这个 agent 上，工具调用权限设为自动。

### 3. 用一个轻量调度脚本发起巡检

在目标服务器上设一个 cron 任务（或其他调度器），每 5 分钟执行一个脚本，向 OpenClaw 的 API 发送一条固定消息，触发巡检 Agent：

```bash
# check_and_act.sh
RESPONSE=$(curl -s -X POST "http://openclaw-host:port/api/agent/invoke" \
  -H "Content-Type: application/json" \
  -d '{
    "agent_id": "system_patrol",
    "message": "执行一次巡检"
  }')

ACTION=$(echo "$RESPONSE" | jq -r '.reply | fromjson? | .action')

if [ "$ACTION" = "alert" ]; then
  SEVERITY=$(echo "$RESPONSE" | jq -r '.reply | fromjson | .severity')
  MSG=$(echo "$RESPONSE" | jq -r '.reply | fromjson | .message + "\n" + .suggestion')
  # 调用企业微信 Webhook 或其它通知
  curl -X POST "$WECOM_WEBHOOK" \
    -H "Content-Type: application/json" \
    -d "{\"msgtype\":\"text\",\"text\":{\"content\":\"[$SEVERITY] $MSG\"}}"
fi
```

这里的关键设计是：**调度脚本不直接判断阈值，而是完全把“感知+决策”交给 OpenClaw Agent 通过 MCP 工具完成**。调度的唯一职责是触发循环和分发结果。这样就把变化频繁的阈值逻辑和提示词调优集中在了 Agent 配置里，而不是散落在脚本里。

### 4. 可选：用 MCP 实现行动闭环

如果需要更复杂的行动（比如自动重启服务、增删防火墙规则），可以把执行动作也抽象成一个 MCP tool，例如 `execute_remediation`，由 Agent 在判断需要时直接调用，外部脚本只需做“触发+兜底通知”。这样整个 proactive 链条就完整闭环了，调度脚本可以瘦身到只有三四行。

## 踩坑点

经过几轮落地，下面这几个点是反复踩过的：

- **上下文状态污染**：同一个 agent 如果被频繁调用，注意每次请求的 `session_id` 是否独立。如果沿用同一个会话，历史指标数据会进入上下文，造成过时判断。建议每次巡检使用新的会话，或通过系统提示强制“只依据当前工具返回结果判断”。
- **JSON 解析鲁棒性**：模型偶尔会在 JSON 前后加 markdown 代码块标记。调度脚本里使用 `fromjson?`（jq 的忽略错误）只做基础防护，更稳健的做法是在脚本中做一次正则清洗，或者让 Agent 输出一个特殊标记包裹的 JSON，再用 `grep -Po` 提取。
- **工具调用超时与重试**：MCP server 读取系统指标如果因为 psutil 卡住（比如挂载点无响应），会导致整个巡检超时。做好 MCP server 侧的 timeout 处理，并在调度脚本里设定 curl 超时和重试逻辑。
- **权限最小化**：如果允许 Agent 调用 `execute_remediation` 这类工具，要严格限制可执行命令的白名单和影响范围，避免一条错误决策直接搞挂生产环境。初期可以先只开放通知能力，跑稳一周后再逐步放开自动修复。

## 可复用建议

这套模式可以平移到很多场景，不限于服务器监控：

- **定时报告**：cron 触发→Agent 通过 MCP 查数据库/API→生成摘要→推到飞书群。
- **条件触发**：结合文件系统事件（inotify）→发消息给 OpenClaw→Agent 判断是否需要处理（如日志异常检测）。
- **人不在时的代办**：日历事件开始前 10 分钟→Agent 自动拉取上下文信息（上次会议纪要、相关文档链接）→发送到手机提醒。

一些工程化建议：

1. **Agent 配置即代码**：把巡检 Agent 的系统提示和 MCP 挂载配置纳入 Git 管理，改动有迹可循。
2. **决策日志**：记录每次巡检的原始响应和最终动作，方便回溯误报/漏报。
3. **灰度开启自动执行**：先跑 silent 模式（只记录不执行），确认决策稳定后再接入真实动作。
4. **用 feature flag 控制 proactive 开关**：出现异常时可以一键关闭所有自动化，退回到纯人工模式。

## 总结

Proactive 能力不是把 AI 助手变成失控的“自动机器人”，而是通过简单的调度层，把已有的感知和推理能力从“等待调用”变成“按需触发”。Cron + MCP + Agent 这条链路的优势在于：**不需要改动 OpenClaw 核心，不重写工具逻辑，就能让现有资产长出主动性**。对于已经在用 OpenClaw 接了大量工具和数据的团队来说，这种增量式改造完全是工程上可落地、风险可控的方案。

希望这个实践的思路，能帮你把躺在聊天框里的助手，真正拉到运维一线去干活。

---

