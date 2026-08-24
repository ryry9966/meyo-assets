---
title: OpenClaw 的 cron vs heartbeat：两种定时任务怎么选
feedId: 34525
source: 综合讨论
publishedAt: 2026-08-24
---

# OpenClaw 的 cron vs heartbeat：两种定时任务怎么选

## 背景
在 OpenClaw 里做自动化，定时触发是绕不开的。无论是定时拉取数据、生成日报，还是给 Agent 做保活和状态巡检，都会遇到两个配置项：`cron` 和 `heartbeat`。很多实践者一开始会把它们都当成“定时器”，但实际接进插件或 MCP 后，经常出现任务重复跑、时区不对、长任务堆积等问题。原因是这两种调度模型根本不同。

## 问题
`cron` 是日历驱动，回答“到哪个时间点执行”。`heartbeat` 是节奏驱动，回答“每隔多久驱动一次”。两者不是同一层级的选择。如果拿 `heartbeat` 去做每天 9 点的报表，系统重启后可能错过固定时间；如果拿 `cron` 去做 30 秒一次的保活，表达式会变得很奇怪，而且最小粒度也不一定支持。

## 做法/步骤

先判断任务属性：

- 绝对时间：每天、每周、每月固定时刻 → `cron`
- 相对周期：每 N 秒/分钟检查、保活、扫描 → `heartbeat`

一个简单配置示例：

```yaml
tasks:
  daily_report:
    type: cron
    cron: "0 9 * * *"
    timezone: Asia/Shanghai
    handler: report.daily

  session_keepalive:
    type: heartbeat
    interval: 30s
    timeout: 10s
    handler: session.ping
```

主流程建议：

1. 任务入口统一加锁。`cron` 任务如果执行时长超过触发间隔，下一次可能又进来，导致并发写同一份文件或重复发消息。用 Redis 锁或本地文件锁，保证同一时间只有一个实例在跑。
2. 每个任务生成 `run_id`，记录 `last_run` 和 `next_run`。这样无论是 cron 还是 heartbeat，出问题都能快速判断是没触发，还是触发后失败了。
3. heartbeat 里不要放长阻塞操作。比如在 heartbeat 里同步调用外部 HTTP 接口，超时设置不合理，会把整个调度循环拖住。轻量检查、异步处理更合适。

## 踩坑点

1. **时区问题**。容器默认 UTC，配置 `cron: "0 9 * * *"` 以为是中国时间 9 点，实际是 UTC 9 点，差 8 小时。必须在任务或全局配置里显式声明 `timezone: Asia/Shanghai`。
2. **cron 和 heartbeat 同时配置同一个 handler**。有人为了“保险”两个都开，结果同一任务被两个调度源触发，出现双跑。二选一，或者用同一个锁来兜底。
3. **cron 的最小粒度**。如果 OpenClaw 的 cron 实现只支持标准 5 段式，就不要试图用 cron 模拟秒级任务。秒级需求用 heartbeat 或独立 worker 更合适。
4. **heartbeat 不是精确时钟**。事件循环繁忙时，触发时间会漂移。对时间精度敏感的任务不要依赖 heartbeat 作为准点触发器。
5. **任务堆积**。heartbeat 的 interval 设置 30s，handler 实际跑 60s，会造成任务排队。需要设置 `timeout` 和 `skip_if_running`，或者把慢任务挪到独立队列。

## 可复用建议

一个简单选型表：

| 维度 | cron | heartbeat |
|---|---|---|
| 触发依据 | 日历时间 | 固定间隔 |
| 适合任务 | 日报、周报、批处理 | 保活、巡检、状态同步 |
| 精度 | 分钟级为主 | 秒级可配 |
| 重叠风险 | 高 | 中 |

建议所有定时任务都用统一包装函数：入口记录 `run_id`、获取锁、设置最大执行时长、退出时写日志。这样无论底层是 cron 还是 heartbeat，行为都可预期。上线前用 dry-run 或手动触发一次，确认时区和锁逻辑。

## 总结
cron 和 heartbeat 不是“哪个更好”的问题，而是“任务需要哪种时间语义”。日历对齐的任务用 cron，节奏驱动的任务用 heartbeat。把锁、时区、超时和日志做好，比纠结调度名词更重要。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/aedbb339e70c3986.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/085139e6f6fd27ce.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/b36f523f0226ebf4.png)

