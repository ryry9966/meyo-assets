---
title: OpenClaw 定时任务选型：cron 按点跑，heartbeat 按节奏跑
feedId: 34338
source: 综合讨论
publishedAt: 2026-08-23
---

## 背景

在 OpenClaw 里接上 Agent 或插件后，很多任务不再手动触发，而是需要周期性执行：每天 9 点发晨报、每 15 分钟做一次健康检查、每小时拉取一次队列深度。

OpenClaw 同时提供两种定时能力：`cron` 和 `heartbeat`。它们都能安排任务，但触发语义完全不同。如果只图配置简单混着用，很容易出现“任务看着在跑，但不是重复执行就是漏跑”的情况。

## 问题

`cron` 是日历触发：它解决的是“在什么时间点做”。  
`heartbeat` 是间隔触发：它解决的是“每隔多久做一次”。

一个典型错误是：需要每天 9 点整发日报，却用 `heartbeat` 设置 `24h`。进程启动时间稍有变化，触发时间就会漂移。反过来，每 15 分钟巡检一次的任务，如果硬写成 `cron: "*/15 * * * *"`，在 OpenClaw 重启后可能触发补跑，造成启动瞬间任务堆积。

所以选型的关键不是“哪个简单”，而是任务依赖的到底是墙钟时间，还是相对频率。

## 做法/步骤

### 1. 先做任务分类

配置前先判断：

- 日历时间任务：固定每天/每周/每月某时点执行，例如每日 9 点发晨报、每周一生成周报。用 `cron`。
- 相对节奏任务：只关心每隔多久做一次，不关心具体墙钟时间，例如每 15 分钟健康检查、每 5 分钟拉取队列深度。用 `heartbeat`。
- 混合任务：需要“每 30 分钟执行，但只在工作时间”，优先用 `cron` 表达式限制小时范围，而不是用 `heartbeat` 加条件判断。

### 2. cron 配置示例

```yaml
schedules:
  - name: cron-daily-digest
    cron: "0 9 * * *"
    timezone: Asia/Shanghai
    catch_up: false
    task:
      ref: digest_agent
      input:
        window: 24h
```

这里有两个关键配置：

- `timezone`：不写时区的话，很多环境默认按 UTC 计算，国内任务会晚 8 小时。
- `catch_up: false`：关闭补跑。OpenClaw 重启后，如果开启补跑，可能会把停机期间错过的任务一次性补完，造成任务洪峰。

### 3. heartbeat 配置示例

```yaml
heartbeats:
  - name: hb-health-check
    interval: 15m
    run_on_start: false
    max_overlap: skip
    timeout: 10m
    task:
      ref: health_check_skill
```

关键配置：

- `interval: 15m` 表示每 15 分钟执行一次。
- `run_on_start: false` 防止 OpenClaw 启动时立即触发一次。多副本部署时，这项能避免启动风暴。
- `max_overlap: skip` 表示上一次任务还没跑完时，新周期直接跳过，而不是继续堆叠并发。
- `timeout` 是单次任务的最大执行时间，超时会被标记失败或终止，避免僵尸任务占用调度。

### 4. 验证方式

不要直接上生产。先 dry-run 或手动触发一次，观察任务能否正常引用到 Agent/插件。然后打开调度日志，重点看：

- `last_run`：上次真实执行时间
- `duration`：单次耗时
- `exit_code`：是否正常退出
- `overlap_skipped`：是否有跳过记录

如果 `duration` 接近或超过调度间隔，就要考虑加长间隔、拆分任务或启用 `max_overlap`。

## 踩坑点

1. **时区默认不一定是本地时区**  
   `cron` 任务如果不显式写 `timezone`，很容易在 UTC 上跑。国内用户要固定写成 `Asia/Shanghai`。

2. **重启补跑导致任务堆积**  
   很多调度系统为了“不丢任务”，默认会在重启后补跑错过的 `cron`。OpenClaw 里如果有类似 `catch_up` 或 `misfire` 策略，建议生产任务显式设为 `false` 或限制补跑数量。

3. **heartbeat 不等于低精度 cron**  
   想用 `interval: 24h` 模拟“每天 9 点”不可靠。只要 OpenClaw 重启、休眠或系统时钟调整，触发时间就会漂移。固定时间点任务必须用 `cron`。

4. **长任务重叠**  
   任务执行耗时超过间隔时，`cron` 和 `heartbeat` 都可能出现重叠。前者可能并发执行同一任务两次，后者会不断积压。务必设置 `max_overlap: skip` 或外部锁。

5. **短 heartbeat 日志膨胀**  
   每 1 分钟或每 5 分钟跑一次的心跳任务，如果每次输出详细日志，几天后日志量会很大。建议只记录错误或摘要，并设置日志保留策略。

## 可复用建议

**选择规则**

| 场景 | 选择 | 关键配置 |
|---|---|---|
| 每天 9 点发晨报 | cron | `timezone`, `catch_up: false` |
| 每 15 分钟健康检查 | heartbeat | `run_on_start: false`, `max_overlap: skip` |
| 每小时拉取数据 | heartbeat | `interval: 1h`, `timeout` |
| 工作日 10 点同步 | cron | `cron: "0 10 * * 1-5"` |
| 每 30 分钟但仅工作时间 | cron | `cron: "*/30 9-18 * * *"` |

**命名规范**

给定时任务加前缀，便于一眼区分触发语义：

- `cron-daily-digest`
- `cron-weekly-report`
- `hb-health-check`
- `hb-queue-watch`

这样后续排障时，看到 `hb-` 就知道它不绑定具体墙钟时间，看到 `cron-` 就知道要查时区和补跑策略。

**监控指标**

生产环境至少记录这四个字段：

- `last_run`
- `duration`
- `exit_code`
- `overlap_skipped`

如果有条件，把定时任务的失败率、最长耗时、跳过次数接入告警。不要等任务完全静默了才发现问题。

## 总结

OpenClaw 的 `cron` 和 `heartbeat` 不是同一种东西的两种写法，而是两种触发语义：

- 任务绑定具体时间点，用 `cron`。
- 任务只关心执行频率，用 `heartbeat`。

选型正确之后，再把时区、补跑、启动触发、重叠策略这些参数显式写清楚，定时任务才能长期稳定运行。定时任务最怕不是“跑不起来”，而是“看起来在跑，但时间不对、重复跑、静默堆积”。这些都可以通过明确调度语义和少量关键配置避免。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/ab8b4145f26ff53c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/e42b7dda45f18041.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/1fe4676a512a637c.png)

