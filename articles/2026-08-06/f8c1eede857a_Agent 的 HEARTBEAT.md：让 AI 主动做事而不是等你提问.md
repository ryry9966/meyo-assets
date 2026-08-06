---
title: Agent 的 HEARTBEAT.md：让 AI 主动做事而不是等你提问
feedId: 31874
source: 综合讨论
publishedAt: 2026-08-06
---

## 背景

大部分 Agent 工作在“提问—回答”模式里：你丢一条指令，它执行、返回结果，然后沉默。这种被动形态在实时监控、周期性同步、日报生成等场景下会显得很笨重，因为总要人去记住“该问了”。

有些团队会把定时脚本和 Agent API 拼在一起，但维护成本高，且破坏了 Agent 原生工具链的上下文。我们真正需要的是一份声明式配置，让 Agent 自己拥有心跳——定期自我触发、执行预设任务，而不改变现有 Prompt 的编写习惯。

## 问题

在没有专用调度服务的前提下，如何让一个 OpenClaw Agent 具备轻量级的自主执行能力？  
常见挑战：

- 不想额外引入 Airflow/Temporal 等重调度器  
- 希望心跳任务与 Agent 工具、MCP 能力深度集成  
- 需要控制成本（token 消耗、API 调用频次）  
- 上下文污染：被动问答与主动心跳共用同一会话时容易混乱  

## 做法：HEARTBEAT.md

我们采用一个约定文件 `HEARTBEAT.md`，放在 Agent 工作区根目录。Agent 在启动时读入，将其中的任务清单注册到内部定时器，之后按照声明自动向自己发起新的“内部指令”。

### 1. 文件结构

```markdown
# Agent Heartbeat

## schedule
| name          | cron          | prompt_template                                  | output  |
|---------------|---------------|--------------------------------------------------|---------|
| daily-report  | 0 9 * * 1-5   | 请分析过去24小时 data/errors.log 并生成摘要报告    | slack   |
| health-check  | */30 * * * *  | 检查所有服务的 /health 端点，异常时通知            | console |
| sync-git      | 0 */4 * * *   | 执行 git pull --rebase 并汇报变更                  | log     |
```

- `name` 仅用于日志标识  
- `cron` 使用 5 字段 cron（分 时 日 月 周），统一用 UTC  
- `prompt_template` 是 Agent 将收到的新指令，支持变量 `{time}`, `{name}` 等  
- `output` 定义结果投递方式，可以是内置通知、MCP 通道或文件  

### 2. 让 Agent 加载 HEARTBEAT.md

最简单的做法是利用 OpenClaw 的 `init` 钩子或一个专门的心跳插件。  
以 OpenClaw 插件为例（伪码）：

```
# 在 openclaw.yaml 中
plugin:
  heartbeat:
    file: HEARTBEAT.md
    timezone: UTC
```

如果不想写插件，也可以把心跳逻辑放进 Agent 的 System Prompt 的顶层 meta 指令里，例如：“每 30 分钟请自行查阅 HEARTBEAT.md 并执行到期任务”。但这种方式缺乏严格定时，不推荐用于精确场景。

### 3. 执行模型

插件读到 `HEARTBEAT.md` 后，为每一条任务创建一个独立会话（或每次执行 fork 新上下文），避免主动心跳污染主对话。  
流程：

1. 调度器触发任务 `daily-report`  
2. 使用 `prompt_template` 构造完整 Prompt  
3. Agent 在该独立会话中运行，允许调用已配置的工具（文件读写、Slack、Git 等）  
4. 结果按 `output` 策略返回  

独立会话可以设置较小的 `max_tokens` 和低的温度，保持可预测性。

### 4. 结果投递

- **slack**：需要 Agent 访问 Slack Webhook 工具（提前在 MCP server 或插件中配置）  
- **console**：将终态结果打印到 Agent 运行日志  
- **log**：追加到本地文件，方便后续 debug  
- **mcp**：通过 [MCP 通知接口](https://modelcontextprotocol.io/) 写给外部系统  

## 踩坑点

1. **时区与 cron 差异**  
   cron 默认 UTC，但是业务时间常为 Asia/Shanghai。务必在配置中强制用户设 UTC 偏移，或者在插件内做转换，否则“早9点报告”可能变成凌晨 1 点触发。

2. **重复执行**  
   如果 Agent 重启时没有持久化上次执行时间戳，可能会在启动瞬间重复执行所有任务。解决：文件里维护 `last_run` 字段或使用外部状态（小型 SQLite）记录。

3. **工具权限泄漏**  
   心跳任务自动调用工具，可能触发写操作。务必限制 heartbeat 会话的工具白名单（例如只能读文件、发送通知，不能删除资源）。

4. **上下文长度爆炸**  
   若任务复杂度高，每次执行会消耗大量 tokens。建议为心跳任务设定单独的小上下文窗口（例如 4000 tokens），任务超出则截断。

5. **失败循环**  
   某条心跳持续报错时，盲目重试会产生大量无意义调用。需设置最大重试次数（如 3 次）以及冷却期。

## 可复用建议

- **将 HEARTBEAT.md 纳入版本管理**，与 Agent 代码一起迭代  
- **提取通用模板**，不同项目只需修改 cron 和 prompt  
- **增加自监控心跳**：另起一个 `heartbeat-watchdog` 任务，定期检查心跳插件自身是否存活  
- **输出标准化**：让所有心跳结果包含执行时间、状态码、摘要，便于接入告警系统  
- **安全存储凭证**：Slack Token、Git 密钥等不要明文写在 HEARTBEAT.md 中，应用 Agent 已有的 Secret 管理

## 总结

从“等人问”到“主动干”，Agent 的行为质变只需要一个声明式文件。HEARTBEAT.md 不依赖重型调度框架，把定时、任务描述和输出策略统一在 Agent 能直接消费的格式中，让自动化变得更整洁。实际落地时注意隔离会话、控制权限和失败策略，就能在很小成本下获得一个可靠的自主 Agent 辅助器。

---

