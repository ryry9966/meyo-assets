---
title: 让 AI 助手“提前干活”：OpenClaw 里可落地的 proactive 链路
feedId: 33131
source: 综合讨论
publishedAt: 2026-08-14
---

最近在 OpenClaw 上把一部分助手从“被动回复”改成“主动执行”，比如每天早上检查 GitHub Actions 失败项、依赖安全公告、磁盘水位。最明显的变化不是“更智能”，而是很多事不再需要我打开对话窗口。

## 背景

现有 Agent 大多是 request-response：你发消息，它运行。但大量运维、监控、信息聚合类任务天然适合 proactive——定时触发、事件触发，或者条件满足后主动通知。

在 OpenClaw/MCP/插件体系里，这件事可以拆成一条链路：

**触发器 → 决策 Agent → 工具执行 → 通知/确认**

不需要引入额外平台，关键是把触发、权限、输出三件事控制住。

## 问题

proactive 难在不是“会主动”，而是“可控”。主要面临三个问题：

1. 什么时候触发？定时、webhook，还是状态变化？
2. 哪些动作允许？直接写文件、发命令，还是只读检查？
3. 如何避免打扰和循环？每次都通知会疲劳，事件互相触发会跑飞。

## 做法 / 步骤

### 1. 先选只读任务

不要一上来做自动修复。建议从这几个开始：

- 每天早上检查 CI 失败任务
- npm/pip 依赖过时或安全公告
- 某个 API 健康检查
- 日志里最近 1 小时错误数

这些任务有一个共同点：**只读、结果明确、适合结构化输出**。

### 2. 建立触发源

OpenClaw 侧可以用 cron 或 webhook 作为入口。定时任务只调用一个“入口 agent/skill”，并传入少量上下文：

```yaml
proactive:
  - name: morning_ci_check
    schedule: "0 8 * * 1-5"
    trigger_type: cron
    entry: ci_check_agent
    tools: [read_ci_failed, read_repo_list]
    notify: ["slack#ops"]
    max_output_tokens: 800
```

这里只是示意，实际配置按你的 OpenClaw 版本来。重点是：**入口要小，不要传历史长上下文**。

### 3. 给工具做白名单

不要让 proactive agent 拿到通用 shell 或写文件能力。通过 MCP server 暴露窄接口，例如：

- `get_failed_workflows(repo, branch, since)`
- `list_outdated_deps(project)`
- `check_disk(mount_point)`

写操作单独放一个 agent，或者需要二次确认。这样即使模型判断失误，也不会直接造成破坏。

### 4. 约束输出与通知

要求 agent 输出固定结构，比如：

```json
{
  "has_issue": true,
  "top_issues": ["repo-a: build failed"],
  "suggestion": "查看 workflow 日志第 3 步"
}
```

通知渠道只接收异常。正常情况可以只写日志，或者每周汇总一次。否则每天“一切正常”很快会被忽略。

### 5. 人工闭环

如果 proactive agent 建议修复，不直接执行，而是生成“待确认任务”。用户回复 yes 后，再触发第二个写操作 agent。这样 proactive 负责发现和报告，执行仍保留人工确认。

## 踩坑点

- **上下文膨胀**：每次 proactive 跑新会话，不要复用历史长上下文。只带 `trigger_type`、时间、必要参数。
- **误操作**：曾经让一个 agent 从“整理文件”变成“移动文件”，结果把暂存目录移走了。后来改成“只读 + dry-run + 人工确认”。
- **通知疲劳**：只通知异常，正常结果写日志；或者用去重键避免同一个问题每小时报一次。
- **循环触发**：一个事件处理后又产生新事件。必须设置 `max_steps` 和去重键，比如用 issue ID 或 commit hash。
- **时区和重试**：定时任务明确 timezone，失败重试要防止重复执行。

## 可复用建议

1. **从“只读摘要”开始**，不直接做自动修复。
2. **工具接口尽量窄**：每个 MCP tool 只做一个动作，参数显式，不搞“智能体自由发挥”。
3. **要求结构化输出**：JSON 或固定字段，后续过滤、通知、去重都方便。
4. **所有 proactive 运行留痕**：记录 trigger、输入、输出、工具调用，便于回溯。
5. **报告 Agent 和执行 Agent 分开**，避免一个 agent 既观察又修改。

## 总结

proactive 不是让模型“更主动”，而是让它在一个受限的时间、工具、输出框架下提前执行。OpenClaw 里最实用的 proactive 形态，其实是：

**定时触发 + 只读 MCP + 异常通知 + 人工确认执行**

先做到“少打扰、能回放、可撤销”，再谈后面更复杂的自动化。

---

