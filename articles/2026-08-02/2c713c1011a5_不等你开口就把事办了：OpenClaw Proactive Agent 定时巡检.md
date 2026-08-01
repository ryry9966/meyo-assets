---
title: 不等你开口就把事办了：OpenClaw Proactive Agent 定时巡检与自愈的工程实践
feedId: 31269
source: 综合讨论
publishedAt: 2026-08-02
---

## 背景

在实际运维、数据巡检、质量监控等场景里，用 LLM 搭建的 Agent 大多仍以被动对话模式工作——等用户在聊天框里输入“帮我查一下错误率”，它才调用工具并返回结果。这种交互模式在需要**持续感知**和**提前干预**的场景下完全不适用。我们希望 Agent 能像靠谱的 SRE 一样，定时扫一眼关键指标，发现异常后立刻做出决策，而不是等老板在群里@才动。

OpenClaw 社区近几个版本对 tool call、MCP 连接器以及 headless 运行的支持逐渐成熟，这让“主动式 Agent（Proactive Agent）”的落地成为可能。下面我们用一个具体案例说清楚怎么把这件事做稳。

## 问题拆解

需求非常直接：监控集群中某个 Deployment 的 CPU 使用率，如果连续 5 分钟超过 80%，就自动扩容到 4 个副本，同时发一条 Slack 告警。整个过程无需人工触发，Agent 要自己按节奏运行、调工具、做决策。

要满足此需求，设计上必须解决三个关键点：

1. **调度**：怎么让 Agent 定时“醒来”？
2. **感知**：Agent 怎么拿到外部实时数据？
3. **安全执行**：自动操作系统资源，必须防止误操作与死循环。

## 实现步骤

### 1. 工具准备（基于 MCP）

让 Agent 能访问 Prometheus 和 Kubernetes API，最利落的方式是把这些后端能力包装成 MCP 工具。OpenClaw 内置了 MCP 工具桥接，可以直接写配置文件：

```yaml
mcp_servers:
  - name: prometheus
    transport: stdio
    command: ["mcp-prometheus", "--url", "http://prometheus:9090"]
  - name: kubectl
    transport: stdio
    command: ["mcp-kubectl"]
```

这样 Agent 就能直接调用 `query_range` 和 `scale_deployment` 这类工具。Slack 通知则用 OpenClaw 自带的 `notify.slack` 插件完成，不用额外封装。

### 2. 定义 Proactive 任务

在 OpenClaw 里，Proactive 任务可以通过一个专用的“巡检 Agent”配置实现。创建一个 `proactive-agent.yaml`：

```yaml
agent:
  name: cluster-guardian
  system_prompt: |
    你是一个自动巡检 Agent。你的任务是检查 Pod CPU 是否健康并做决策。
    - 你必须使用 prometheus 工具查询最近 6 分钟的 CPU 均值。
    - 如果连续 2 个采样点（当前和 3 分钟前）都超过 80%，则判定为异常。
    - 异常时，先调用 kubectl scale 将 deployment/backend 扩容至 4 副本，再调用 notify.slack 发送告警。
    - 若未满足阈值，只记录info日志，不执行任何操作。
    - 绝对禁止在没有数据的情况下做出变更。
  tools:
    - prometheus/query_range
    - kubectl/scale
    - notify.slack
  max_iterations: 5
```

这里的 system prompt 刻意写明了决策条件和安全边界，避免 LLM 自由发挥。

### 3. 定时触发

Agent 本身不包含定时器，因此调度交给系统的 crontab（或者 Kubernetes CronJob）。最简单的方式是每 3 分钟执行一次：

```bash
*/3 * * * * openclaw run --agent proactive-agent.yaml --headless --log-file /var/log/ocl/proactive.log
```

`--headless` 让 Agent 以非交互模式运行，执行完工具调用后自动退出。日志落盘，方便回头审计：是什么触发了扩容、Agent 推理链路是什么。

### 4. 防抖与幂等

连续两次 cron 触发可能因为前一次还未退出而导致重复扩容。解决方案很简单：在脚本中添加文件锁：

```bash
#!/bin/bash
exec 9>/var/lock/ocl-proactive.lock
if ! flock -n 9; then
  echo "Previous run still active, exiting."
  exit 0
fi
openclaw run ...
```

此外，扩容工具需要带上当前副本数校验，避免从 4 再扩到 4。

## 踩坑点

- **上下文残留**：Agent 在每次运行间状态不互通是好事，但若需要带一点点记忆（比如“上次检测到过但没连续超标”），可以借助 MCP 读写一个小型 KV 存储或直接基于 Prometheus 历史数据二次计算，不要强行用 long-running agent。
- **幻觉导致误扩**：例如 Prometheus 返回空结果时，LLM 可能“推断”出 CPU 100% 的危险值。必须在 prompt 中强制要求：**数据为空则视为正常，不采取动作**。
- **权限最小化**：通过 MCP 暴露的 kubectl 工具务必限制 RBAC，只允许 scale 特定 Deployment，绝不赋予 cluster-admin。
- **超时问题**：如果 MCP 连接 Prometheus 卡住，Agent 会阻塞整个 cron 运行。给 `openclaw run` 加上 `--timeout 30s` 防止僵尸进程堆积。

## 可复用建议

把“主动巡检”能力抽象为可复用的模式：

1. **巡检技能包**：将“查询指标→阈值判断→动作执行→通知”做成一个 OpenClaw Skill，类似函数一样可以用 `skill: proactive-check` 调用，方便部署到不同服务。
2. **决策与执行分离**：让 Agent 只输出决策建议（diagnosis）和要调用的工具，再通过沙盒 runner 二次确认后执行。这在生产环境尤其重要。
3. **事件驱动升级**：如果条件允许，可以结合 Webhook 触发器（例如 Alertmanager 发过来的告警）直接启动 Agent，可减少探测定时延迟。
4. **回放与调试**：将所有 tool call 的入参、返回记录到结构化日志，方便离线回放 Agent 的决策过程，而不是在黑盒里猜。

## 总结

完全被动的 AI 助手只能解决“等一下，我帮你查”级别的问题。结合定时调度、MCP 化的基础设施工具和明确的决策约束，OpenClaw 的 Agent 完全可以做到“问题还没报上来，它就已经修完并通知到位了”。这种 proactive 能力并不是银弹，需要花精力做好边界条件、异常处理和人工兜底，但在降噪、缩短 MTTR 上的收益肉眼可见。

---

