---
title: OpenClaw 里做 proactive agent：从“等指令”到“条件触发”的工程化笔记
feedId: 33763
source: 综合讨论
publishedAt: 2026-08-18
---

# OpenClaw 里做 proactive agent：从“等指令”到“条件触发”的工程化笔记

## 背景

现在大部分基于 OpenClaw、MCP 和 Agent 的实践仍然是 turn-based：用户发一条消息，Agent 跑一轮工具调用，返回结果。这个模式在对话、查询、代码辅助场景里够用，但放到运维、研发流程、内容监控里就很吃力。比如“证书还有 7 天过期”“某个 issue 挂了 5 天没人回”“凌晨构建失败但没人看到”，这些事如果等用户开口再处理，往往已经错过了最佳处理窗口。

Proactive 能力的核心不是让 Agent 没事自己乱跑，而是把“检测 - 判断 - 执行 - 通知”这个闭环从被动响应，前移到条件触发。真正工程化的 proactive 系统，应该像一个后台巡检进程，行为可预期、边界清晰、失败可追溯。

## 问题

直接让 Agent 定时全量跑会带来几个典型问题：

- **误报和噪声**：条件过宽，每天往群里推一堆“建议关注”的消息，最后没人看。
- **权限失控**：Agent 拿到写权限后，可能因为 prompt 不够收敛，把“提醒”变成“自动改配置”。
- **上下文膨胀**：长时间运行的 proactive session 会把历史工具结果都塞进上下文，越跑越慢，判断也越来越飘。
- **静默失败**：定时任务没跑、工具挂了、通知发了但没人处理，比没做 proactive 更危险，因为它给人“已经有人管了”的错觉。

所以 proactive 不能简单等于“加个 cron 调 Agent”。

## 做法 / 步骤

我在自己项目里用的是一套“触发源 + 策略文件 + 受限 Agent + 审计日志”的组合，实践下来比较稳。

### 1. 选择触发源

优先用事件订阅，而不是轮询。比如 GitHub 的 webhook、数据库的 change stream、消息队列。只有在第三方不提供事件推送时，才用 cron 轮询作为兜底。

在 OpenClaw 里，我一般配置一个单独的 headless session，例如 `proactive-github-watch`，用系统 cron 或内置调度器触发：

```text
*/10 * * * * openclaw run --session github-watch --message "check stale issues"
```

这个 session 与日常对话 session 隔离，避免污染用户历史。

### 2. 用策略文件约束“什么时候能动手”

不要把所有判断都交给 Agent 的 system prompt。判断条件最好落在结构化策略里，Agent 只负责执行和生成自然语言摘要。一个精简的 YAML 示例：

```yaml
triggers:
  - name: stale_waiting_issue
    source: github
    schedule: "0 9 * * 1-5"
    condition:
      state: open
      label: waiting
      inactive_days: ">= 7"
    action:
      tool: github_comment
      template: "Hey @{{ assignee }}, this issue has been waiting for 7 days. Is there anything blocking?"
    limit:
      max_actions_per_run: 3
      idempotency_key: "stale_{{ issue_id }}_{{ date }}"
    notify:
      channel: ops-im
      level: info
```

这样每次 run 最多处理 3 个 issue，且同一天不会对同一个 issue 重复评论。条件变更时改 YAML 即可，不用重新调 prompt。

### 3. 把动作白名单交给 MCP

Agent 只暴露三个 MCP 工具：

- `github_list_issues`：只读查询
- `github_comment`：可逆操作，但需要明确模板
- `notify_ops`：发通知到 IM

不允许直接调用 shell、不允许打开浏览器、不允许执行代码。这样即使 Agent 判断失误，最坏结果也就是发一条评论或一条通知，不是灾难。

### 4. 审计和记录

每次 proactive run 生成一个 `run_id`，把触发条件、匹配到的对象、执行的动作、结果都写进 SQLite 或 JSONL。这样一周后可以回溯：哪些是真的有用，哪些是误报，有没有重复动作。

## 踩坑点

### 轮询频率与 API 限额

GitHub、Notion、Linear 这类服务都有 rate limit。10 分钟一次全量查询，很快就会撞限。我后来改成先查 `updated_at` 变化，再拉详情，rate limit 压力小很多。如果服务支持 webhook，直接用事件推送，别轮询。

### 确认机制不是万能的

有人会加一个“高危动作需要人工确认”。但如果 proactive 任务在半夜触发，没人点确认，任务就卡住了。更合适的做法是：高危动作永远只生成“待办清单”，而不是执行；只有低风险、可逆动作才自动跑。

### 时区和夏令时

cron 表达式很容易在服务器时区和用户时区之间出错。建议所有 schedule 统一用 UTC，展示层再做本地化。别用 `0 9 * * *` 以为是中国时间早上 9 点，服务器可能在 UTC。

### 静默失败必须报警

proactive 任务最怕的不是报错，而是不报。给调度器加一个 watchdog：如果某个 task 超过预期时间没跑，或者连续 3 次 run 都没产出结果，就主动发一条告警。否则你以为它在默默守护，其实它已经挂了两周。

## 可复用建议

- **分级触发**：L1 只读报告，L2 可逆操作，L3 高风险动作只生成审批清单。绝大多数 proactive 场景用 L1/L2 就够。
- **幂等键**：任何写操作都带 `idempotency_key`，防止重复执行。
- **影子模式**：新策略先跑一周“只记录不执行”，对比人工实际操作，确认误报率和漏报率再放开。
- **通知收敛**：不要把每一次 proactive run 的结果都推给全员。按对象或类型聚合，例如一天一条摘要，紧急事项走单独通道。
- **给 Agent 设 TTL**：proactive session 不要无限积累历史，每次 run 后裁剪上下文，只保留当前对象和策略信息。

## 总结

Proactive 能力不是让 Agent 变“主动”，而是让它在一个受控、可审计、可回滚的框架里，替你在条件满足时先走一步。它的价值不在“快”，而在“稳定地不漏”。从触发源、策略约束、工具白名单到审计日志，每一层都在降低误动作和静默失败的概率。等这套东西跑顺之后，你才会真正感受到：好的 Agent 不是话多，而是该出现的时候出现，不该动的时候保持安静。

---

