---
title: 让 AI 助手“主动”起来：在 OpenClaw 中构建 Proactive Agent 的工程实践
feedId: 31082
source: 综合讨论
publishedAt: 2026-07-31
---

## 背景

绝大多数 AI 助手遵循“请求-响应”模式：你说一句，它动一次。但在工程场景里，大量价值恰恰来自“你还没开口，它已经把事情办好了”的能力——比如监控到异常自动触发诊断、代码仓库有新 PR 时主动生成代码审查摘要、日历事件前自动准备会议材料。

这种 proactive（主动/预判式）能力，本质上不是模型更强，而是 **触发逻辑 + 工具链 + 安全护栏** 的工程组合。OpenClaw 作为多 Agent 框架，自带的调度器、MCP 工具集成和记忆模块，恰好给 proactive Agent 提供了比较干净的土壤。本文用一次实际构建，聊聊如何在不引入过度复杂性的前提下，让 Agent 学会“抢答”。

## 问题

Proactive 并不是让 Agent 随意乱动。需要解决的工程问题至少包含：

1. **时机判断**：何时该出手？基于时间、事件还是状态变化？
2. **上下文保鲜**：怎么避免拿着过期信息做错事？
3. **副作用控制**：主动行为一旦出错，影响远大于被动响应。
4. **可观测性**：悄悄干完用户不知道，等于没干。

我们把场景收敛到：监控一个 GitHub 仓库的近期 Issue，每天早上 9:00 自动生成一份 Issue 摘要，通过企业微信机器人推送到群聊。

## 做法与步骤

### 1. 搭建触发骨架

OpenClaw 支持通过内置的 `scheduler` 能力配置定时任务。直接在 Agent 的配置里声明 cron 表达式即可，无需外接 crontab 容器。

```
scheduler:
  - name: morning-issue-digest
    cron: "0 9 * * 1-5"
    task: generate_issue_digest
```

这里的 `generate_issue_digest` 是一个自定义任务名，之后会在 Agent 的 system prompt 中关联。

### 2. 接入工具（MCP）

用 GitHub MCP Server 暴露一套 issue 查询工具。在 OpenClaw 的 MCP 客户端配置中添加：

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "<your-token>"
      }
    }
  }
}
```

同时接入企业微信机器人的 webhook 作为通知通道，可以直接用一个简单的 HTTP tool 封装。

### 3. 编写 Proactive Agent 的 system prompt

关键在于把“定时触发”和“任务执行”的关系说清楚，并约束行为边界。示例片段：

```
你是 Issue Digest Agent。
每天上午 9:00 你会自动收到 generate_issue_digest 任务。
当收到该任务时，请执行以下步骤，且不得主动执行其他与仓库修改相关的操作：

1. 使用 list_issues 工具获取仓库 owner/repo 最近 24 小时新建及更新的 issue。
2. 仅提取标题、发起人、标签前 3 个、正文前 200 字。
3. 汇总为一句话摘要 + 3 条最值得关注的 Issue 详情。
4. 将结果通过 send_wework_message 发送到群聊，格式为：
   - 🗂 今日 Issue 速览（日期）
   - 一句话摘要
   - 1. [标题] (链接) - 重点原因
5. 完成后输出简短 log。
```

这里没有用“主动监控”这种模糊词，而是把主动触发完全交给调度器，Agent 只负责响应调度事件，从而降低行为不确定性。

### 4. 增加状态记忆防止重复推送

如果因为某些原因任务在同一天被重复触发，Agent 会再次推送相同摘要。解决方式是写入一条“今日已推送”的记录。OpenClaw 的记忆模块可以存 key-value，我们用日期做 key：

```
在执行推送后，调用 memory_set 函数，key 为 "last_digest_date"，值为当天日期。
任务开始时，先 memory_get("last_digest_date")，如果等于今天且十分钟内已推送过，则直接结束，不重复发送。
```

这样把幂等逻辑下沉到工具调用层面，而不是让模型“记住”，更可靠。

## 踩坑点

**频率与上下文污染**  
刚运行时总觉得 Agent “不太主动”，于是把 cron 改成每 5 分钟一次。结果是，每次触发都会带上之前的执行记录，上下文窗口快速膨胀，模型开始混淆任务意图。最终保持每天一次，且每次执行后清空 session 历史，只保留记忆模块中的幂等标记。

**主动行为必须“可逆思维”设计**  
如果 Agent 除了发摘要还能修改 issue 标签，那 proactive 就变成高风险操作。踩过坑后，我们把 Agent 的工具权限严格限定为只读 + 通知，任何写操作都交给另一个需人工确认的 Agent。

**时区与调度精度**  
OpenClaw 调度器默认使用 UTC 时间，第一次配置 cron 时按本地时间写，结果在半夜收到了“早上”的摘要。需要在容器环境中显式设置 TZ 环境变量，并做好注释。

## 可复用建议

从这次实践中抽象出一个简单的 proactive 模式，适用于大多数非关键路径的主动任务：

- **Sensor（传感器）**：由调度器、webhook 事件或状态变更回调充当，不依赖模型判断“何时动”。
- **Decider（决策器）**：Agent 根据 system prompt 定义的任务流，决定是否执行、如何执行，并用记忆模块存储状态，避免重复决策。
- **Actuator（执行器）**：通过 MCP 工具或插件完成具体动作，权限收拢到只读/可通知/可回滚的操作。

在 OpenClaw 中，Sensor 用 scheduler 或 event listener 实现；Decider 用 Agent + memory；Actuator 用 MCP tool set。三者解耦后，可以像搭积木一样拼出各种“主动”场景，而不必每次都担心 Agent 失控。

另外，如果团队对 proactive 有更高要求，可以考虑给 Agent 增加一个“意图列表”（intent list），每个意图明确触发条件、执行次数上限、回滚策略。这能让主动能力从一次性脚本进化成可维护的系统能力。

## 总结

让 AI 助手“不等你开口就把事办了”，听上去很智能，但在工程上是可控的组合：**确定性触发 + 清晰任务流 + 最小权限工具 + 幂等与记忆**。OpenClaw 提供了不错的基座，剩下的工作是把“主动”关进合适的笼子里，让它只在规定的时间、规定的地点，做规定的事。

Proactive 不是玄学，是一种对“按时干活”的工程承诺。

---

