---
title: OpenClaw 定时任务二选一：cron 与 heartbeat 的工程化取舍
feedId: 31835
source: 综合讨论
publishedAt: 2026-08-06
---

## 背景：一个调度器，两种时间语义
OpenClaw 的插件体系里，Scheduler 是最容易被低估的组件。它负责在合适的时间触发你的 Agent、脚本或 MCP 工具链。官方内置了两种触发器：**cron** 和 **heartbeat**。表面看都能实现“定时执行”，但底层的时间语义完全不同。用错场景，轻则告警抖动，重则漏掉关键自动化，还没人知道原因。

这篇文章不讨论谁更好，只从真实踩坑的角度，讲清楚什么时候选什么、怎么配置、以及出问题时该从哪排查。

## 问题：选择焦虑的根源
我在一个混合环境中同时维护了数据备份、健康检查、过期缓存清理、Agent 存活监控四条自动化流程。最初偷懒全用 cron，很快暴露两个问题：
- 健康检查每 5 分钟跑一次，但服务在两次 cron 之间挂了，要等满 5 分钟才报警。
- Agent 存活监控靠 cron 去轮询心跳端点，十几台 Agent 时任务开始堆积，而且当 Agent 自身时钟漂移，cron 的“定时”毫无意义。

后来换成 heartbeat 触发，又遇到新坑：备份任务改成“每次心跳后触发”，导致备份频率失控；清理任务因 Agent 短时离线而没触发，残留文件撑爆了磁盘。

问题的本质是：**你究竟在等待一个时刻，还是在等待一个信号？**

## 做法：配置层面的差异
OpenClaw 的触发器配置都在 `scheduler.triggers[]` 里，核心字段如下。

**cron 触发器**  
适用于“每 N 分钟/小时/天”这种周期性、与外部状态无关的任务。  
```yaml
triggers:
  - name: daily-report
    type: cron
    expression: "0 2 * * *"
    timezone: "Asia/Shanghai"
    task: report-gen
```
关键参数：
- `expression`：标准 cron 表达式
- `timezone`：务必显式设置，默认 UTC 会让凌晨 2 点的任务跑到上午 10 点

**heartbeat 触发器**  
适用于“当某个 Agent 还活着/刚刚恢复/心跳丢失”这类依赖外部事件的任务。  
```yaml
triggers:
  - name: guard-duty
    type: heartbeat
    source: agent-42
    missed_threshold: 3
    on_missed: alert-escalation
    on_recovery: clear-alert
```
关键参数：
- `source`：心跳信号来源，可以是 Agent ID 或 MCP 服务端点
- `missed_threshold`：连续丢失几次心跳才触发，避免瞬时抖动
- `on_missed` / `on_recovery`：对丢失和恢复分别挂载不同的任务

更进阶的用法：heartbeat 还可以跟 MCP 资源状态联动。比如某个 `mcp://db/connection_pool` 的资源一旦变为 `exhausted`，心跳触发扩容脚本，而不需要 cron 频繁轮询。

## 踩坑点：生产环境真实教训
**1. cron 的时区陷阱**  
OpenClaw 调度器运行时强制使用容器内 `/etc/localtime`，即使 YAML 里写了 `timezone`，如果镜像没装 `tzdata`，会静默回退到 UTC。请在 Dockerfile 里加：
```
RUN apk add --no-cache tzdata
ENV TZ=Asia/Shanghai
```
并在调度器日志中确认第一条执行时间。

**2. heartbeat 的“假阳性”与“假阴性”**  
Agent 因网络抖动偶尔丢一个心跳，`missed_threshold: 1` 会引发告警风暴。设成 3 次以上比较安全，但需要评估：三次心跳间隔 × 阈值的总时长是否已经超出你的故障容忍度。  
另一个极端：Agent 进程僵死但不退出，仍旧发送心跳，heartbeat 触发器以为一切正常。因此在 `on_recovery` 中不能只依赖心跳恢复，建议额外执行一个轻量健康探针（比如 `mcp call health-check`）作为恢复条件。

**3. 任务重叠与幂等性**  
cron 的常见陷阱：一次任务没跑完，下一个 cron 周期又启动了。如果任务本体没做锁，会产生竞争。OpenClaw 的 Scheduler 提供了 `concurrency_policy: forbid` 来阻止重叠，但会丢弃后续触发。更稳妥的方法是任务内部使用数据库锁或文件锁，确保幂等执行。  
heartbeat 触发如果源 Agent 心跳恢复过于频繁（比如网络闪断反复），会在短时间内连续触发 `on_recovery`，造成“雪崩”。加上 `min_interval_seconds: 60` 参数可以限制触发频率。

**4. 调度器本身的可用性**  
无论 cron 还是 heartbeat，调度器自身宕机都会导致所有任务静默停止。cron 任务可以事后补跑（手动或通过 `catch_up` 策略），但 heartbeat 丢失就是永远丢失。生产环境务必对调度器进程做守护，并将心跳触发事件写入持久化日志，便于事后审计。

## 可复用建议
整理一个简单的决策路径：
- 任务依赖“绝对时间”，且对延迟不敏感（分钟级可接受）→ 用 cron。
- 任务依赖“外部系统状态变化”，或需要秒级响应 → 用 heartbeat。
- 监控类场景：heartbeat 做存活检测，cron 做趋势聚合报告。
- 如果不得不用 cron 轮询状态，考虑把轮询间隔缩短到比 heartbeat 失联容忍时间更短，但会增加系统负载。
- 混合使用：heartbeat 丢失后触发的告警任务，可以由 cron 定时做汇总和降噪，避免告警淹没值班人员。

配置模板可以用一个“防御性 YAML”清单自查：
- cron: 检查 `timezone`、`concurrency_policy`、`catch_up`
- heartbeat: 检查 `missed_threshold`、`min_interval_seconds`、恢复任务的健康探针
- 公共：所有任务必须有幂等保障和超时设置

## 总结
cron 与 heartbeat 没有高下之分，而是两种时间抽象。cron 相信墙上的时钟，heartbeat 相信系统的心跳。理解这个区别，你的自动化流程才不会被“定时”两个字误导。下次拆解需求时，先问一句：这件事是因为到了某个时间点该做，还是因为某个信号表明该做？答案会指引你选对触发器，也能让排查问题时少走弯路。

---

