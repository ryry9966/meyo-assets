---
title: Heartbeat 即指令：用 HEARTBEAT.md 让 Agent 从被动应答变成主动值守
feedId: 31864
source: 综合讨论
publishedAt: 2026-08-06
---

## 背景：Agent 的“主动”从来不是标配

当前大多数 Agent 应用的交互模式仍然是“用户提问 → Agent 响应”。即便是挂载了 MCP 服务器的智能体，也只是在接到用户指令后才去调取外部资源、生成回答。如果你想让 Agent 每天早上 8 点自动汇总 GitHub issues，或者在监测到某个 API 状态异常时主动推送通知，只靠对话式触发完全不够。

工程上，我们需要一种更接近守护进程（daemon）的心跳机制——Agent 定期自检一份“指令清单”，根据时间、条件或外部事件自主执行任务，而不是等人开口。这就是 **HEARTBEAT.md** 的由来：一份人类和 AI 都能读写的 Markdown 指令文件，作为 Agent 主循环的心跳入口。

## 问题拆解

一个典型的“主动值守”场景包含三个核心诉求：

1. **定时触发**：按 cron 表达式执行任务（例：每个工作日 9:00 生成当日站会摘要）。
2. **条件触发**：当外部条件满足时自动动作（例：当 `staging` 分支有新 PR 合并时通知群聊）。
3. **状态自持**：Agent 能记住“已经做过什么”，避免重复执行产生噪音。

社区里已有的做法要么是将定时能力硬编码在 Agent 框架中，要么是通过外部 cron job 调用 Agent 接口。前者耦合度高，后者把“指令”和“触发策略”分在了两个系统里，调试和维护都很痛苦。HEARTBEAT.md 把合二为一：指令文件既是触发清单，也是执行日志。Agent 只需要一个 loop——“读取文件 → 解析计划 → 执行 → 写回状态”即可。

## 做法与步骤

### 1. 定义 HEARTBEAT.md 的格式

为了让 Markdown 既能被人轻松阅读，又能被 Agent 可靠解析，采用 YAML front matter + 任务列表的组合：

```markdown
---
heartbeat: "*/5 * * * *"
agent: openclaw
notify:
  - discord-webhook: https://discord.com/api/webhooks/xxx
  - local: /tmp/agent-heartbeat.log
---

# Heartbeat Plan

- [ ] **每工作日 09:00** 汇总过去 24h 的 GitHub issues 并推送到 Discord
  - schedule: "0 9 * * 1-5"
  - task: github-issues-summary
  - when: last_run_before > 24h
- [ ] **当 API 健康检查失败 3 次时** 发送报警
  - condition: api-health-check.failure_count >= 3
  - task: send-alert
  - throttle: 30m
- [ ] **每 6 小时** 拉取天气数据写入本地 CSV
  - schedule: "0 */6 * * *"
  - task: fetch-weather
```

front matter 定义全局心跳间隔 (`heartbeat`) 和通知通道。每个任务条目用 checklist 项描述，Agent 解析时会按照 `schedule` 或 `condition` 字段决定是否执行。

### 2. 实现 Agent 的心跳循环

以 OpenClaw 生态为例，可以直接写一个最小化的 runner 脚本，用 Node.js 实现：

```js
import { parseHeartbeat } from '@openclaw/heartbeat-parser';
import { executeTask } from './tasks.js';
import fs from 'fs/promises';

const HEARTBEAT_PATH = './HEARTBEAT.md';

async function loop() {
  const raw = await fs.readFile(HEARTBEAT_PATH, 'utf-8');
  const plan = parseHeartbeat(raw);   // 解析 front matter + checklist 项

  for (const task of plan.tasks) {
    if (shouldRun(task)) {
      const result = await executeTask(task);
      await appendLog(task, result);
      // 更新 HEARTBEAT.md 中对应项的状态（如打勾）
      await markTaskDone(HEARTBEAT_PATH, task.id);
    }
  }

  // 按 heartbeat 间隔再次调度
  setTimeout(loop, plan.heartbeatMs);
}

loop();
```

`shouldRun()` 会检查任务的时间满足性、条件表达式以及节流控制。`executeTask` 则根据任务类型调用对应的 MCP 工具或内部能力，例如 `github-issues-summary` 实际会通过 MCP 的 GitHub provider 拉取 issues 并喂给 LLM 生成汇总，再由 `notify` 通道发出。

### 3. 对接 MCP 做条件触发

HEARTBEAT.md 中的 `condition` 本质上是指向 MCP 资源状态的查询。例如 `api-health-check.failure_count >= 3` 需要 Agent 在每次心跳时调用健康检查 MCP server 获取当前失败计数。实现时可以把这部分抽象成一个状态缓存：每次循环先调用 MCP 拉取关键指标，存入内存 K-V，再在条件判断时直接从缓存取值，避免重复请求。

状态缓存的好处是可以支持更复杂的条件，例如“在过去 15 分钟内失败次数 > 2”，也方便跨任务共享数据。只需记得在心跳间隔较长时，缓存不要过久，设置 TTL 为 heartbeat 的两倍即可。

## 踩坑点

1. **Markdown 解析的一致性**  
   AI 阅读 Markdown 时对缩进、空行很宽容，但程序解析很严格。使用统一的 YAML front matter 是关键，避免在正文里直接写非结构化指令。团队内部可以维护一个 JSON Schema 来校验 heart beat plan，每次修改后由 CI 检查。

2. **任务状态写回导致无限触发**  
   最常见的问题是执行完任务后写入“已完成”标记，但下一次心跳又认为该任务仍待运行。解决方案：给每个任务生成唯一 `id`（基于指令行的 hash），并在 HEARTBEAT.md 底部维护一个最近执行日志区块，比如 `<!-- heartbeat:log -->`，Agent 每次运行前先检查日志内的“最近完成时间 + 节流窗口”。

3. **外部条件轮询成本**  
   如果 `condition` 依赖外部 API（如天气、健康检查），而心跳间隔设得过短，很可能打爆 API 限额。建议对条件检查单独设定 `check_interval`，比如心跳 2 分钟，但健康检查仅每 10 分钟执行一次。在 HEARTBEAT.md 中用 `check: "every 10m"` 字段明确区分。

4. **并发执行**  
   如果通过系统 cron 同时启动了多个 Agent 实例，且都试图写 HEARTBEAT.md，会产生竞态。最好在 runner 启动时创建锁文件（如 `HEARTBEAT.lock`），如果锁已存在则直接退出。更稳健的做法是用文件系统原子写入，或使用 SQLite 存放执行状态，但对于一个 Markdown 方案，锁文件足够。

## 可复用建议

- **模板即规范**：为团队提供一个 HEARTBEAT.md 模板仓库，包含常见场景（定时报告、条件报警、数据归档），团队成员只需修改 front matter 中的 `schedule` 和 `notify` 字段即可复用到自己的项目。
- **与 OpenClaw workflows 集成**：如果已经在用 OpenClaw 的工作流插件，可以把 HEARTBEAT.md 作为工作流的唯一触发器，工作流步骤再按任务类型分发到不同的执行器。这样可以利用 OpenClaw 内置的对话记忆和 MCP 客户端，减少自研工作。
- **本地测试先行**：在正式接入 Discord、飞书等通知通道前，先把 `notify` 通道设为 `local`，Agent 把执行日志写到本地文件，开发者可以 `tail -f` 观察行为，直到符合预期再上线。
- **MCP 状态提供者独立部署**：将健康检查、数据拉取等能力封装成独立的 MCP server，HEARTBEAT.md 只写条件表达式，不关心具体实现细节。这样更换监控对象时改 HEARTBEAT.md 即可，不用动 Agent 代码。

## 总结

HEARTBEAT.md 不是一个银弹产品，而是一种轻量级的 Agent 自主性实践模式。它将定时、条件执行和状态管理下沉到一份人类和 AI 共享的文件中，让 Agent 拥有持续“呼吸”的能力。对于已经在使用 OpenClaw 或 MCP 的团队，只需一个简单的 runner 脚本，就能把被动助手变成项目里默默干活的“心跳守护者”。

当你明早醒来时，看到 Discord 里已经静静躺着一份今日待办和代码仓库动态汇总，正是 HEARTBEAT.md 在午夜悄悄完成了它的工作——这，才是 Agent 该有的样子。

---

