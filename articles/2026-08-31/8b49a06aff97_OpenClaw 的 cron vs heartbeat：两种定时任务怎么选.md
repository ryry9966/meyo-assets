---
title: OpenClaw 的 cron vs heartbeat：两种定时任务怎么选
feedId: 35525
source: 综合讨论
publishedAt: 2026-08-31
---

## 背景

在 OpenClaw 里给 Agent、MCP 工具或插件做定时任务时，通常会遇到两个触发器：cron 和 heartbeat。它们看起来都能“定时执行”，但语义差别很大。选错后常见症状是：任务偶尔不跑、重复跑、半夜被唤醒、或任务列表里出现积压。

## 问题

cron 是墙钟时间语义：在指定时间点触发，例如每天 09:00、每周一 02:00。heartbeat 是运行间隔语义：进程或调度器每隔固定时长 tick 一次，例如每 30 分钟、每 5 秒。

关键区别不是“能不能定时”，而是“按世界时间对齐，还是按运行节奏对齐”。这个区别会直接影响任务在重启、阻塞、补跑场景下的表现。

## 做法/步骤

1. 先判断任务是否依赖真实时间。  
   - “每天 9 点发日报”“工作日 18 点同步”选 cron。  
   - “每 30 分钟刷新缓存”“上次跑完后再过 10 分钟重试”选 heartbeat。

2. 检查进程生命周期。  
   OpenClaw 服务重启后，cron 下一次触发取决于当前时间；heartbeat 通常从启动时间重新计数。如果 heartbeat 需要跨重启保持节奏，应持久化 last_run，并在启动时判断是否补跑或跳过。

3. 处理重叠与漂移。  
   cron 任务执行超过下一触发周期时，可能再次进入，需加锁或 skip_if_running。heartbeat 若阻塞事件循环，间隔会漂移；不要用 heartbeat 替代高精度定时器。

4. 配置上建议显式声明 timezone 和 jitter。  
   cron 如 `0 9 * * 1-5` 要确认时区，避免在 UTC 下跑错。heartbeat 可加 0-5% jitter，降低多实例同时拉取造成的峰谷。

## 踩坑点

- cron 在进程宕机期间不会补跑。关键任务不要默认“每天至少一次”，应增加启动补偿检查。
- heartbeat 不是“每秒保证一次”。Agent 内部有长任务、模型调用或网络等待时，tick 会被延后。
- 两者都可能重复触发。幂等性必须做在任务内部：写入前查重、使用唯一键、避免无状态重复调用外部 API。
- 短 heartbeat（如 5 秒）容易与 MCP 工具超时叠加，造成资源占用；先评估任务耗时，再设置间隔。

## 可复用建议

- 墙钟边界用 cron：报表、定时发布、营业时间动作。
- 运行期自愈用 heartbeat：缓存刷新、状态巡检、连接保活、退避重试。
- 组合使用：cron 负责“什么时候开始”，heartbeat 负责“开始后多久再检查一次”。
- 小型 Agent 任务统一配置一个 scheduler 入口，避免在多个插件里各起一套 heartbeat，导致难以观察。
- 记录 last_run_at / next_run_at / last_result 三个字段，排障时先看“该跑没跑”还是“跑了但失败”。

## 总结

cron 适合与真实时间对齐的一次性触发，heartbeat 适合与运行状态对齐的周期性循环。OpenClaw 里没有绝对好坏，只有是否匹配任务语义。先把“按点跑”和“隔一段时间跑”区分清楚，再处理时区、漂移、重叠和幂等，定时任务就不会变成定时事故。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/8fe735cdaa6bf450.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/f075cf9788b58d91.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/f69f28664e25c077.png)

