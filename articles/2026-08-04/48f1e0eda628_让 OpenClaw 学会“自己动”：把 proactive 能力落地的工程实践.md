---
title: 让 OpenClaw 学会“自己动”：把 proactive 能力落地的工程实践
feedId: 31529
source: 综合讨论
publishedAt: 2026-08-04
---

## 背景：我们说的 proactive，不是闹钟

社区里关于“主动”的讨论越来越多，但大多数场景其实是“定时任务”，不如说那是被动里的被动。真正的 proactive 是：agent 在用户没有发出请求的情况下，基于环境状态变化，自主判断“该做事了”，并且自己完成它。

我们在 OpenClaw 里跑通了一套相对克制、可复现的 proactive 链路。不玄学，核心就三件事：感知什么、凭什么判断、怎么兜底。

## 问题：伪主动与真主动的分界线

OpenClaw 本身是 event-driven 的框架，这给了实现 proactive 的底层可能性。但一开始我们走了弯路：把“定期轮询然后执行”当主动，结果只是加了一个 cron。真正的主动需要具备三个要素：

1. **非请求触发**：不是用户说“去做”，而是外部事件或状态偏离触发。
2. **自主决策阈值**：agent 自己判断“现在是否值得打扰你”。
3. **可撤销/可追溯**：主动做的事出错时，要有恢复路径。

做不到最后一条，主动只会添乱。

## 做法：在 OpenClaw 里搭一条最小 proactive 链路

我们的实现方案分四层，全部基于 OpenClaw 原生的 MCP 和 memory 机制，不依赖特定插件。

### 1. 环境感知层（替代“用户开口”）

在 MCP server 里注册一个 watcher 工具，比如监听一个目录、一个 RSS、一个 GitHub issue 列表。重点是：**用订阅替代轮询**。OpenClaw 的 MCP 支持 server push 事件通知，尽量用这个能力，省资源而且时效性好。

如果上游不支持 webhook，退而求其次用低频率轮询，但要把状态 hash 存进 memory，避免每次全量处理。

### 2. 判断执行层（agent 的“自主性”）

这是最容易做砸的地方。我们的做法是把判断逻辑从“给 LLM 一段自由发挥的 prompt”改成**结构化决策矩阵**：

```json
{
  "trigger": {
    "type": "repo.issue.opened",
    "target": "owner/repo"
  },
  "decision": {
    "priority_rule": "if label contains 'bug', then priority=high",
    "action_rule": {
      "high": "create_branch + draft_pr + comment",
      "low": "collect + weekly digest"
    }
  }
}
```

决策矩阵可以放在 OpenClaw 的 rules 目录里，被 agent 启动时加载。这么做的原因是：proactive 的本质是“在没指令的情况下做出决策”，如果决策逻辑完全交给模型，行为不可控，成本也高。规则 + 模型的组合最务实：规则负责判断“要不要动”，模型负责“怎么动得像个正常人”。

我们用了一个小技巧：在 prompt system 里显式声明 `you may take actions without user request only when all conditions in the action_rule are met`。这看起来简单，但对抑制模型“过度积极”非常有效。

### 3. 执行层（MCP 工具编排）

OpenClaw 的 MCP 工具层是天然的执行端。我们把上次提到的 mcp.surf 思路延伸了一下：每个 proactive 动作都是一个 MCP 工具链，比如 `[git.create_branch] -> [git.commit] -> [gh.pr.create]`。

关键点是**工具链要有幂等性**。主动执行和被动执行不同，没有人在中间确认，“再执行一遍会怎样”必须自回答。我们的做法：开始前先查状态（PR 是否已存在），存在就跳过并记录“already_done”。

### 4. 反馈与收敛层（怎么不变成骚扰）

每次 proactive 动作之后，把结果写回 OpenClaw memory：

```
action: pr_created
trigger: issue #42
embrace: success
user_reaction: silent (no complaint within 24h)
```

下个周期调整决策矩阵参数。比如连续两次没被用户点赞的动作，自动降级为“只通知不执行”。

## 踩坑点：这些坑我们替你踩了

**1. 自激循环**。agent 主动创建了一个 issue，watcher 又监听到这个 issue，于是再次触发……直到把 API quota 烧完。解决方式：在触发事件里过滤 agent 自身产生的行为，方法是给所有 agent 操作打一个 `x-agent-action: true` 的标记。

**2. 过度敏感**。第一版把 `file modified` 当触发源，结果日志文件每秒变动一次，agent 每 10 秒就“主动”汇报一次。后来加了静默区（silence window）：同一个触发源在 30 分钟内只响应一次。

**3. 幻觉“主动”**。模型会在没有触发事件时，脑补出一个“用户可能想要”的场景然后执行。例如我们曾发生过 agent 在深夜自动把 staging 分支合进了 main。此后加了硬性 guardrail：`main` 分支的任何操作必须有人工确认 token，这个逻辑写在 OpenClaw 的插件层而非 LLM 层。

**4. 上下文丢失**。多次 proactive 动作之后，agent 会忘记自己以前做过什么。我们最终在 memory 里建立了一个 `actions_log` 命名空间，每次读写都走这个固定 key。不要依赖对话历史来记这些，太脆。

## 可复用建议

给想在 OpenClaw 上做 proactive 的社区朋友几条实在建议：

- **别一上来就做全自动**。先跑“只建议不执行”模式跑一周，积累用户对“什么被主动”的容忍度数据。
- **决策矩阵放在配置里，不要写死在代码里**。你会经常调阈值的，尤其当“打扰率”上升时。
- **所有主动动作必须留痕**。不是为了审计，是因为调试时你根本不知道它为什么动。OpenClaw 的 plugin 钩子可以做统一记录。
- **可选的人工确认开关**。在低风险动作上设 auto，高风险动作上设 confirm。这是“主动”和“越权”的平衡点。
- **复用社区已有的 MCP server**。主动能力的关键不在写很多新工具，而在于把已有工具串成有状态的事件驱动链。

## 总结

OpenClaw 已经给了足够的地基。proactive 能力在工程上不是“更智能的模型”，而是**三层决策的清晰分离**：环境规则负责感知、策略矩阵负责响应、模型负责表达与润色。再加上强制的幂等性、留痕机制和人工 guardrail，proactive 才能从一个好听的 demo 变成一个可控的生产能力。

一句话：先让它学会“不做什么”，再让它学会“主动做什么”。前者永远比后者重要。

---

