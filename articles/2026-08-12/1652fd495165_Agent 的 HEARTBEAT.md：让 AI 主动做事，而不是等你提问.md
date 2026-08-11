---
title: Agent 的 HEARTBEAT.md：让 AI 主动做事，而不是等你提问
feedId: 32653
source: 综合讨论
publishedAt: 2026-08-12
---

# Agent 的 HEARTBEAT.md：让 AI 主动做事，而不是等你提问

## 背景

大多数 Agent 系统（尤其是基于 OpenClaw 搭建的助手、MCP 工具链、自动化流程）的典型工作模式是**请求-响应**：你发指令，Agent 干活，然后等待下次指令。这种模式在小规模、短生命期的场景下很自然，但一旦我们把 Agent 变成**常驻服务**——比如让它长期监控某个数据源、维持一个上下文会话、定时执行复杂工作流——问题就出现了：

- Agent 进程可能已经卡死、OOM、上下文溢出，但你完全不知道，直到下一次手动提问时才踩坑。
- 外部依赖（API、数据库、MCP Server）挂了，Agent 不会主动告诉你，只能被动地等到任务失败时才暴露。
- 上下文累积过多后，模型推理质量下降，但没有任何自净机制，只能靠人工重启。

我们需要的不是更频繁的“人盯着 Agent”，而是让 Agent **自己有一套心跳机制**，定期自我检查、主动汇报状态，就像运维中的 `HEARTBEAT` 信号。

## 问题拆解

要让 Agent 主动心跳，得解决几个工程问题：

1. **何时触发**：Agent 当前正在等待用户输入，如何让它“自己醒来”执行检查？
2. **检查什么**：哪些指标能真实反映 Agent 是否健康，且不引入过大开销？
3. **如何汇报**：结果推送到哪里？如何避免信息噪音？
4. **上下文隔离**：心跳任务与主会话混在同一个上下文里，会不会污染推理？

接下来，我们用 OpenClaw + MCP 的具体实践，一步步搭建一套 **HEARTBEAT.md 自律方案**。

## 做法与步骤

### 1. 定义心跳的检查项：HEARTBEAT.md

在 OpenClaw 项目的 `config/` 或根目录创建一个 `HEARTBEAT.md`，它既是**可读的说明文档**，也是 Agent 在心跳执行时会**解析的规则集**。示例内容：

```markdown
# Agent Heartbeat Configuration

interval: 10m
timeout: 30s
report_channel: mcp::notify::feishu_webhook

checks:
  - id: memory_usage
    type: system
    command: free -m | awk '/Mem/{print $3/$2 * 100}'
    warning_threshold: 80
    critical_threshold: 90

  - id: context_length
    type: internal
    metric: token_count
    warning_threshold: 6000
    critical_threshold: 7500

  - id: mcp_server_health
    type: http
    url: http://localhost:8000/health
    expected_status: 200

  - id: db_connectivity
    type: script
    script: |
      python3 -c "import psycopg2; psycopg2.connect('...').close()"
```

这条文档明确了几件事：检查间隔、每项检查的命令或端点、告警阈值。Agent 可以直接读取这个 Markdown 结构（借助结构化解析或简单的前置指令），从而知道"该查什么"。

### 2. 用定时触发器让 Agent“自己醒来”

OpenClaw 原生支持通过插件机制挂载定时任务，或者借助 MCP 社区工具（例如 `mcp-server-cron`）来触发 Agent 逻辑。最简单的方案：

- 利用 OpenClaw 的 `task` 机制，在项目配置中增加一个隐藏的周期性 Task。
- 或者使用 MCP 的 `scheduled_tasks` 能力，在 MCP Client 端以 `cron` 表达式驱动一个轻量级 Agent。

示例 OpenClaw 配置（`openclaw.yaml` 片段）：

```yaml
tasks:
  - name: heartbeat
    schedule: "*/10 * * * *"
    action: run_heartbeat
    input:
      config_file: "./HEARTBEAT.md"
    isolated_context: true
    max_tokens: 200
```

关键点是 `isolated_context: true`，这样心跳检查会在一个**全新的、极短的上下文**中执行，不会与主会话混在一起，避免污染历史。

### 3. 执行检查并生成报告

心跳 Task 被触发后，Agent 执行以下逻辑（可内置为一个 Skill 或 MCP Tool）：

1. 解析 `HEARTBEAT.md` 中的 `checks` 列表。
2. 顺序执行每项检查：调用系统命令（如 `free`）、内部 token 计数器、HTTP 请求等。命令执行直接通过 MCP 的 `run_command` 或系统插件完成；HTTP 检查可调用 `fetch` 工具。
3. 将结果与阈值比对，生成一个简短的 Markdown 报告，例如：

```
[HEARTBEAT] 2025-03-19 10:30:00
- memory_usage: 78% (WARNING)
- context_length: 5420 tokens (OK)
- mcp_server_health: OK
- db_connectivity: FAIL (CRITICAL)
```

4. 通过配置的 `report_channel` 发送报告。这里用了一个假的 MCP 命名空间 `mcp::notify::feishu_webhook`，实际实现可以是任何消息推送工具。

如果所有检查都 OK，可以选择**不发送报告**，只落地到本地日志，防止信息轰炸。

### 4. 闭环：失败项的自动恢复（可选进阶）

心跳不只是告警，还可以触发简单的自愈动作。例如，当 `context_length` 超过临界阈值时，心跳 Task 可以直接调用 `reset_context` 或清理旧消息的 MCP 工具；当某外部 API 不可达时，可尝试重启 MCP Server。这种自动恢复适合低风险的软故障，但要谨慎设计，避免误判导致数据丢失。

## 踩坑记录

实践过程中有几个典型坑：

- **系统命令超时阻塞心跳**：某些检查（如 `db_connectivity`）可能在网络异常时长时间挂起，导致整个心跳流程卡死。务必为每个检查设置独立超时（如 `timeout: 5s`），并利用 `asyncio` 或多线程防止阻塞。
- **token 计数不准确**：不同模型的 tokenizer 有差异，Agent 内部拿到的是近似值。建议用 model 提供的实际 `usage` 字段，并设置比模型上下文窗口低 10-15% 的告警线，预留缓冲。
- **环境变量泄漏**：通过 MCP 的 `run_command` 执行命令时，容易把敏感环境变量带进日志。务必在命令中避免输出私密信息，并对日志做脱敏处理。
- **定时器与主流程的上下文污染**：即使 `isolated_context` 设计了独立上下文，也要确保心跳任务返回的消息不会作为用户消息注入主会话。可通过特定的消息标识（如 `role: system` 且带 `heartbeat` tag）来过滤。

## 可复用建议

1. **HEARTBEAT.md 即契约**：把检查规则文档化，Agent 直接读取，而不是硬编码在配置里。这样即使不熟悉代码的协作者也能调整阈值。
2. **用 MCP 工具解耦**：所有检查项（系统命令、HTTP、Python 脚本）都封装为 MCP Tool，这样心跳逻辑可以复用到不同的 Agent 实例，且方便测试。
3. **分层告警**：低于 warning 阈值的不告警，warning 发低频摘要，critical 实时推送并加自动恢复尝试。统计一段时间的心跳历史，可以生成 Agent 健康度趋势，提前发现缓慢恶化的问题。
4. **本地先跑通**：先在本地用 `cron` 模拟心跳，简单打印检查结果，确认无误后再接入真实通知渠道。

## 总结

给 Agent 加上 `HEARTBEAT.md` 驱动的自律心跳，本质上是在**系统稳定的前提下，让 Agent 具备最轻量的 OODA（观察-判断-决策-行动）循环**。它不需要复杂的多 Agent 架构，一个小而美的周期性任务就能大幅降低“Agent 悄无声息挂掉”的风险。

关键收获：

- 一份可读的 `HEARTBEAT.md` 作为检查规则源。
- 利用 OpenClaw 的定时 Task 或 MCP 定时触发器，实现独立上下文的心跳执行。
- 执行结果通过推送渠道主动汇报，减少人工巡检负担。
- 注意超时、token 近似、环境隔离等细节。

现在就可以在你的 Agent 项目里加一个心跳文件，让它从被动应答工具，变成会主动报平安的协作伙伴。

---

