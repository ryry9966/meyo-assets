---
title: Agent 的 HEARTBEAT.md：让 AI 主动做事而不是等你提问
feedId: 31964
source: 综合讨论
publishedAt: 2026-08-07
---

## 背景
在 OpenClaw 或类似 Agent 框架里，模型默认是被动响应式的：你问，它答。即便接了 MCP 工具、文件系统、浏览器插件，Agent 依然是“等待指令”的状态。  
但在大量工程场景中，我们希望系统能**按节奏主动检查状态、抓取信息、发出通知，甚至自动修复**——就像一个运行在后台的心跳循环，而不是每次都靠人推一把。

典型的诉求包括：
- 每隔 30 分钟检查 API 是否可通，异常时自动打日志并发送告警
- 每天 8 点拉取仓库最新 issue，按标签分类汇总到群聊
- 监控某个目录变化，一旦有文件更新就自动解析并写入数据库

这类需求无法通过常规的 prompt 实现，因为它们不再是“一次问答”，而是**持续运行的自动化循环**。

## 问题
如何用一种最小化、可复现、不打破现有插件体系的方式，让 Agent 具备“主动做事”的能力？  
不要引入额外调度平台，不要修改框架核心代码，同时保持 **Agent 可观测、可中断、可配置**。

针对这个问题，我们整理出一套以 `HEARTBEAT.md` 为准心的方案。

## 什么是 HEARTBEAT.md
`HEARTBEAT.md` 是一份存放在 Agent 工作区根目录下的配置文件，用来描述 Agent 的周期性任务。  
它不是一个新 DSL，而是一个 Markdown 文件，里面用结构化的 section 声明：
- **心跳间隔**：单位秒或 cron
- **任务列表**：每个任务的名字、描述、调用工具、参数、重试策略
- **触发条件**：除了定时，还可以有文件变化、状态变化等（需配合 watcher）
- **通知策略**：任务结束后是否通知、通知渠道（如日志、webhook）

Agent 启动时读取这份文件，在后台启动一个轻量级 runner，按配置消费任务。

这种做法有几个优势：
- **完全普适**：不依赖任何特定框架特性，只要 Agent 能解析 Markdown 即可
- **渐进式**：不加心跳文件，Agent 行为与原来完全一致；加了就进入主动模式
- **可版本管理**：和代码仓库放在一起，任务变更可追溯、可 review

## 做法 / 步骤
### 1. 创建并维护 HEARTBEAT.md
在 Agent 的工作目录下新建 `HEARTBEAT.md`，示例结构如下：

```markdown
# Heartbeat Configuration

## interval
*/5 * * * *

## tasks
### 1. health_check
- tool: http_get
- params: { "url": "https://api.example.com/health" }
- assert: response.status == 200
- on_failure:
  - action: log_error
  - action: webhook
    params: { "url": "https://hooks.slack.com/xxx" }

### 2. daily_repo_summary
- cron: 0 8 * * *
- tool: open_claw_cmd
- params: { "command": "git pull && python summarize.py" }
- notify: slack_channel
```

### 2. 实现心跳 runner
不要魔改 Agent 主事件循环。可以借助 OpenClaw 的插件机制或 MCP 的 long-running tool，实现一个 `heartbeat_runner` 工具。  
这个工具读取 `HEARTBEAT.md`，解析 interval 和 tasks，启动一个定时器。每个触发时刻：
- 按顺序执行任务
- 捕获工具调用的异常
- 将结果写入 `HEARTBEAT.log` 或推送到通知渠道

如果框架支持 lifecycle plugin，可以在 Agent 初始化时注册一个 `start_heartbeat` 插件，自动加载 runner。否则，在 prompt 中要求 Agent 在第一次对话时手动调用一次 `start_heartbeat`。

### 3. 任务编排与边界
复杂的任务链（如“先检查状态，再发通知，再重试一次”）不宜全写到 Markdown 里，那样会变成另一个 YAML 地狱。  
建议只在 HEARTBEAT.md 里定义**原子任务**，编排逻辑放在对应的 tool 实现里。例如 `daily_repo_summary` 工具内部完成 pull、汇总、推送三步，对外只是一个任务。

### 4. 增加状态与防重
有些任务不能重复执行（如每天发一次汇总）。可以在 runner 里维护一个基于文件或内存的状态表，记录上次执行时间、成功/失败状态，配合 cron 表达式来决定是否触发。  
更稳健的做法是：让工具自己检查状态。例如汇总脚本在运行前先检查今天是否已生成过文件。

## 踩坑点
1. **资源占用**：如果心跳频率过高（如每秒一次），会导致工具调用风暴，消耗大量 token 和 CPU。合理频率建议 ≥ 30 秒，除非是本地轻量检测。
2. **工具超时与重试**：长时间运行的工具可能卡住整个 runner。务必给每个工具调用设置超时，并实现简单的指数退避重试（最多 2 次）。
3. **错误静默失败**：任务在后台失败后，如果既没打日志也没通知，会“假运行”。每一次 on_failure 尽量写文件或触发 webhook。
4. **配置文件热加载**：修改 `HEARTBEAT.md` 后，是否需要重启 Agent？最简单的方案是：每次 runner 定时唤醒时重新读取文件，这样就能在下个心跳周期生效，无需重启。
5. **多 Agent 实例并发**：如果同一工作区有多个 Agent 实例，它们会同时读取同一份 `HEARTBEAT.md` 并执行同样任务。需要引入锁机制（如文件锁、数据库锁）或使用单例 runner。

## 可复用建议
- **将心跳任务抽象成 MCP tools**：配置中只写工具名和参数，工具本身可以在不同 Agent 间复用。例如 `health_check` 是一个标准的 MCP 工具，任何 Agent 的 HEARTBEAT.md 都能直接调用。
- **用日志还原“心跳轨迹”**：在 `HEARTBEAT.log` 里记录每次触发的时间、执行的任务、结果摘要。配合 `grep` 或简单的 dashboard，就能快速判断心跳是否存活。
- **模板化**：维护一个基础 HEARTBEAT 模板仓库，包含常见任务（健康检查、文件备份、数据同步），新项目直接拷贝并按需裁剪。
- **不要过度工程化**：如果只有两三个定时任务，手动写 HEARTBEAT.md 就够了。不用引入 dag、任务依赖等概念，保持纯粹性。

## 总结
`HEARTBEAT.md` 解决了 Agent “动不起来”的问题，用一份存于代码库的 Markdown 文件赋予其主动性。它不需要新增调度中间件，**让 Agent 在工具生态内完成自我周期性驱动**。  
对于正在用 OpenClaw 或 MCP 做自动化的人来说，这套轻量模式可以快速落地：先从一个简单的健康检查任务开始，跑通后再逐步丰富任务列表。最终你会得到一个既听话、又会主动做事的 AI 工作伙伴。

---

