---
title: OpenClaw 定时任务选型：cron 与 heartbeat 的真实差异
feedId: 31806
source: 综合讨论
publishedAt: 2026-08-06
---

## 背景：为什么需要两种定时机制？

在 OpenClaw 的自动化流水线里，Agent、插件或 MCP 服务经常需要按固定节奏执行任务：拉取增量日志、清理过期会话、向外部系统上报心跳、每天生成统计报表。框架内置了两种定时触发模式——**cron** 和 **heartbeat**。不少开发者第一次配置时会随意选择，导致任务执行时间不准、重复触发甚至把服务打挂。本文从工程实践的角度，把两种方式的边界、踩坑点和选型建议拆清楚。

---

## 问题：cron 和 heartbeat 根本不是同类抽象

表面上看都是“定时”，但二者的抽象层级完全不同：

- **cron**：基于自然时间的调度，`0 8 * * 1` 就是“每周一早八点”，它不关心上一次何时结束、框架是否刚重启，只认时间戳。
- **heartbeat**：基于**固定间隔**的周期性回调，由框架以固定频率（如 15 秒）调用你的处理函数。它是“启动后每 X 秒执行一次”，与自然时间无关。

混淆这两类抽象是 90% 定时任务翻车的根源。比如你想“每天凌晨做数据快照”，却用了 heartbeat 并设成 86400 秒，一旦服务重启，快照时间就会偏移。反过来，用 cron 去做“每 30 秒检查还有多少未处理消息”，cron 表达式写不了秒级，只能落到分钟，根本对不上。

---

## 做法：在 OpenClaw 中的实际配置

这里以插件的 `openclaw.config.ts` 为例，展示两种模式的最小可用配置。注意以下代码基于当前社区主流写法，实际请参考你使用的 OpenClaw 版本。

### 1. cron 配置

```typescript
// plugins/cleanup.plugin.ts
export default {
  name: 'cleanup',
  cron: {
    expression: '0 3 * * *',         // 每天凌晨 3 点
    timezone: 'Asia/Shanghai',       // 明确指定时区，默认为 UTC
    options: { runOnInit: false },   // 服务启动时不立即执行
    task: async (ctx) => {
      await ctx.services.session.cleanExpired();
      ctx.logger.info('session cleanup done');
    },
  },
};
```

- **expression**：标准 cron 表达式，支持 5 位（分 时 日 月 周），不秒级。
- **timezone**：强烈建议显式设置，否则默认为 UTC，导致北京时间错误。
- **task**：既可以是异步函数，也可以返回 Promise，框架会 await 结束。
- **options**：常用 `runOnInit` 控制是否在启动时立即执行一次。

### 2. heartbeat 配置

```typescript
// plugins/watchdog.plugin.ts
export default {
  name: 'watchdog',
  heartbeat: {
    interval: 30_000,               // 30 秒一次
    maxConcurrency: 1,              // 最多允许一个实例同时运行
    task: async (ctx) => {
      const health = await ctx.services.mcp.ping();
      if (!health.ok) {
        await ctx.notify('mcp 离线');
      }
    },
  },
};
```

- **interval**：单位毫秒，从上次 task resolve 之后开始计时，而非固定时间表。
- **maxConcurrency**：防止上一个 task 未完成时下一次又触发，设为 1 即串行化，新周期会推迟到前一个完成后才开始计时。
- heartbeat 没有 `timezone` 概念，也不依赖 cron 引擎，用 `setTimeout` 驱动的链式调度。

---

## 踩坑点：来自真实排障记录

### 1. cron 的时区坑

默认时区是 UTC，国内用户写了 `0 8 * * *` 以为早上 8 点，实际上 UTC 8 点对应北京时间 16 点。排错时总是纳闷“日志里明明计划了，怎么下午才跑”。**解法**：永远显式设置 `timezone: 'Asia/Shanghai'`，并在 CI 里做校验。

### 2. cron 的重叠与跳过

如果 task 执行时长超过了表达式间隔（比如每分钟跑一次，但 task 跑了 75 秒），不同实现处理不同。OpenClaw 默认**跳过重叠**：若前一实例还在运行，新的调度会被直接丢弃，不会排队。这意味着任务会**静默丢失**。需要监控任务的执行耗时，必要时改表达式或拆解任务。

### 3. heartbeat 的“随时间漂移”

heartbeat 是“完成后再等 interval”，不是“每隔 interval 固定触发”。若 task 平均耗时 3 秒，间隔设 30 秒，实际触发周期是 33 秒。如果耗时抖动大，累积效果会让每天的触发次数不可预测。不适合“每整分钟拉一次数据”这种精确场景。

### 4. heartbeat 的停止与恢复

当服务被 suspend 或进入 idle（例如冷启动的 serverless），heartbeat 会暂停，恢复后立刻续上计时。如果看重“被打断的时间也要补执行”，heartbeat 做不到，必须依赖 cron 的时间锚点或额外补数逻辑。

### 5. 并发控制不是银弹

`maxConcurrency: 1` 保证了串行，但如果 task 长期卡住，后续所有 heartbeat 都会被无限推迟，导致监控失效。务必结合超时机制（如 `Promise.race` + `setTimeout`）或拆出独立 worker。

---

## 可复用建议

**按场景选型**  
- 需要**自然时间锚点**（每天/每周固定时刻）且执行频率不高于每分钟一次：用 cron。  
- 需要**秒级轮询、心跳上报、健康检查**等固定周期动作，且不要求严格时间对齐：用 heartbeat。  
- 任务耗时可能超过间隔：优先设计为异步解耦（如扔队列），而非依赖 `maxConcurrency` 硬扛。

**监控与告警**  
- 为每个 cron 或 heartbeat 的任务记录 `lastRunAt`、`durationMs`、`status`。  
- 使用 OpenClaw 内置的 `ctx.scheduler.record`（若有）或外部 metrics，超过两次未执行就报警。  
- heartbeat 建议附带“看门狗”逻辑：如果连续 N 次无响应，上报并尝试自动重启依赖。

**多实例部署**  
cron 若在多实例中运行，每份实例都会触发。需要用分布式锁（如 Redis）保证仅一个实例执行。heartbeat 天然多实例并行，除非你要全局唯一心跳，才加锁。通常建议 heartbeat 不加锁，让每个实例独立上报自身健康。

---

## 总结

cron 和 heartbeat 本质上分别解决“在特定时刻做某事”和“每隔一段时间做某事”两类问题。在 OpenClaw 项目里，选错不仅会让任务执行时间失控，还会引入静默跳过、堆积延迟等隐患。配置前先问自己一句：**这个任务的触发依据是墙上的时钟，还是跑表的节奏？** 答案立刻就能指到正确选项。

附上一份快速决策清单，供团队内部评审时使用：

1. 是否需要精确到秒级？ → 是：heartbeat（并接受耗时造成的周期偏移）；否：cron。
2. 是否依赖真实时间（如“0 点切换 T 日”）？ → 是：cron；否：可用 heartbeat。
3. 是否需要在多实例中仅跑一次？ → 是：cron + 分布式锁；否：heartbeat 或多实例都跑。
4. 任务耗时是否会超过定时间隔？ → 是：异步解耦，避免重叠或推迟。
5. 是否服务频繁启停？ → 是：cron 能保证时间点正确，heartbeat 会漂移。

掌握这两者差异，能避免你在凌晨三点被不是 bug 的 bug 闹醒。

---

