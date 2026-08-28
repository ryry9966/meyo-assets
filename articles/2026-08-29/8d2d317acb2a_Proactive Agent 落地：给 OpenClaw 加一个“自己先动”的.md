---
title: Proactive Agent 落地：给 OpenClaw 加一个“自己先动”的触发层
feedId: 35162
source: 综合讨论
publishedAt: 2026-08-29
---

## 背景

在 OpenClaw 这类 Agent 框架里，大多数自动化仍停留在“反应式”：用户说一句，模型调一次工具，任务结束。真正 proactive 的能力，是让 Agent 在没有人开口的情况下，根据事件、时间或状态变化先执行动作，再回来汇报结果。

典型场景并不复杂：

- 凌晨检测到磁盘使用率超过 80%，提前清理旧快照或创建扩容工单；
- 监控到某个 GitHub issue 被标记为 critical，自动在运维群发通知并创建跟踪任务；
- 依赖服务恢复后，主动通知相关人并触发回归检查。

这些场景的本质不是“模型突然变聪明”，而是给 Agent 加一个可靠的触发层。

## 问题

如果把 proactive 理解成“让模型定时随便想想”，通常会踩三个坑：

1. **误触发多**：没有明确策略，模型容易把小波动当成异常；
2. **重复执行**：同一个事件被多个周期重复消费，动作执行多次；
3. **权限失控**：主动动作越过安全边界，直接做了本应人工确认的写操作。

所以工程化落地的重点，不是让模型更主动，而是让触发、判断、动作、审计形成闭环。

## 做法 / 步骤

### 1. 先窄后宽：从固定事件源开始

不要一上来就让模型在长时间窗口里自由发挥。先选定一两个稳定的触发源，例如：

- cron 定时器；
- webhook 事件；
- MCP server 暴露的 resource 变化；
- OpenClaw 插件内部的事件钩子。

触发源越具体，越容易回放和排障。

### 2. 设计四段式 Proactive 管道

```
Trigger Source -> Policy Filter -> Action Executor -> Record / Trace
```

- **Trigger Source**：cron、webhook、MCP subscription；
- **Policy Filter**：用确定性规则或小模型判断，不直接做危险动作；
- **Action Executor**：只执行白名单动作，如发通知、创建任务草案、调用只读 MCP 工具；
- **Record / Trace**：每次触发都写日志，带 trace id 和决策原因。

一个 OpenClaw 插件的伪代码骨架大致是：

```ts
// 伪代码：proactive watcher 插件
const rule = {
  id: "disk-high",
  trigger: { type: "cron", expr: "*/15 * * * *" },
  policy: async (ctx) => {
    const snap = await ctx.mcp.call("infra.snapshot", {});
    const last = await ctx.state.get("last:disk-high");
    return snap.diskUsage > 0.8 && Date.now() - last > 6 * 3600e3;
  },
  action: async (ctx) => {
    await ctx.actions.enqueue("ops.task", {
      title: "disk cleanup",
      severity: "P2",
      source: rule.id,
    });
    await ctx.state.set("last:disk-high", Date.now());
  },
};
```

关键是 `policy` 先做条件过滤，`action` 只做低风险操作。真正要修复磁盘时，先创建任务草案，再由人确认。

### 3. 先跑 Shadow Mode

上线 proactive 规则时，先让 Agent 只记录“我本来会做什么”，不执行动作：

```ts
if (ctx.config.shadowMode) {
  await ctx.log("shadow", {
    rule: rule.id,
    wouldAction: "ops.task.create",
    reason: "diskUsage=0.83",
  });
  return;
}
```

跑两三天，看误报率、触发频率、决策质量，再逐步放开。

### 4. 与 MCP 工具配合

如果外部系统已经暴露 MCP tool，proactive 动作可以直接调 MCP，不必让模型读大段日志。比如监控、查询、创建工单都可以抽象成 MCP 工具。OpenClaw 插件负责触发和策略，MCP 负责读业务状态和执行业务动作，这样边界更清楚。

## 踩坑点

- **重复触发**：事件去重键必须持久化。用 `eventId + ruleId` 做 hash，存在 OpenClaw state 或 Redis 里，设置合理过期时间。
- **静态阈值抖动**：比如 CPU 在 79% 到 81% 之间反复横跳。加冷却时间、最小变化幅度或连续 N 次满足才触发。
- **动作边界过大**：主动动作不要直接删除、重启、发布。只读查询、发通知、创建工单草案相对安全。写操作必须 human-in-the-loop。
- **上下文浪费**：每个周期让模型重新读历史，会快速消耗 token。让 MCP 工具返回摘要或结构化状态，模型只做必要判断。
- **webhook 安全**：外部 webhook 要做签名校验、限流和事件去重，否则容易被刷或重复投递。
- **不可观测**：没有 trace 的 proactive 动作，事后很难说清为什么触发、为什么执行。每次循环记录触发原因、决策条件、执行动作和结果。

## 可复用建议

1. **白名单动作表**：先禁用所有写操作，只允许通知、查询、创建任务草案。
2. **状态分离**：业务状态和 Agent 状态放在 state/存储里，不塞进 prompt，避免上下文膨胀。
3. **全局 kill switch**：一个开关能随时暂停所有 proactive 规则，出问题时先停再查。
4. **事件日志可回放**：把事件流存成日志，支持 dry-run 重放某天的事件，验证规则变更是否合理。
5. **度量误报率**：记录触发次数、实际执行次数、人工取消次数。如果人工取消比例高，说明策略需要收紧。
6. **先通知后修复**：第一阶段只做通知，人确认后再逐步允许自动修复。

## 总结

Proactive 能力不是让模型“更主动”，而是把触发、判断、动作、审计做成一条可靠管道。先选窄场景，跑 Shadow Mode，控制动作边界，记录每次决策。这样 Agent 才能在没人开口时真正把事情办成，而不是制造一堆噪音和误操作。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/e13f79c26bd400fd.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/897e342175c70459.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/70673321f9c2d19b.png)

