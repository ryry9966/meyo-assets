---
title: OpenClaw 定时任务选型：cron 与 heartbeat 的正确使用姿势
feedId: 32831
source: 综合讨论
publishedAt: 2026-08-12
---

## 背景

在 OpenClaw 搭建自动化工作流时，几乎任何略复杂的 Agent 或 MCP 插件都会遇到定时执行的需求——从每小时的报表生成，到持续监听消息队列的消费者。OpenClaw 内置了两种定时触发机制：**cron** 和 **heartbeat**。不少刚上手的同学会混淆两者的适用场景，要么把简单周期任务写成复杂的自旋循环，要么把需要持续保持心跳的守护进程用 cron 粗粒度触发，导致状态丢失或延迟不可控。

简单来说：
- **cron**：基于时间表达式，在预设的时间点执行一次性任务。
- **heartbeat**：以固定间隔周期性触发，通常用来维持长连接、探活或轮询。

本文面向已在使用 OpenClaw 做插件/自动化实践的用户，结合工程经验梳理两种机制的差异、配置方法、容易踩的坑以及可复用的选型建议。

## 问题界定

最近在优化一个 MCP 插件时，我需要同时实现两类逻辑：
1. 每天凌晨 2 点对前一天采集的日志做聚合归档。
2. 监听 Redis Stream，实时消费消息并触发下游 Agent。

第一个显然是 cron，第二个如果用 cron 每分钟跑一次，不仅浪费资源，还会在消费阻塞时留下死信。而 heartbeat 机制恰好适合第二种场景：服务启动后以 5 秒间隔苏醒，轮询 Redis 是否有新消息，处理完后继续休眠。

但是，如果不加约束地使用 heartbeat，很容易陷入“执行时间大于间隔导致任务堆叠”或者“分布式多实例重复执行”的坑。因此，选型不仅仅是“选哪个”，还包括在 OpenClaw 框架内如何安全配置。

## 做法 / 步骤

### 配置 cron 定时任务

在 OpenClaw 的 `.claw.yaml` 或对应的 `TaskDefinition` 中，cron 任务通过标准的 5 位表达式定义。一个典型的每天 2 点归档任务如下：

```yaml
tasks:
  - name: archive-logs
    type: cron
    schedule: "0 2 * * *"
    action: plugin://log-archive/run
    options:
      timezone: Asia/Shanghai
      retry: 2
```

需要注意，OpenClaw 的 cron 调度器默认使用 UTC 时间，如果没有显式设置 `timezone`，经常会出现在国内服务器上凌晨 2 点其实是下午 10 点执行的问题。上述配置中通过 `options.timezone` 强制指定时区，可以避免这种混乱。

cron 任务设计要点：
- 任务应该是**无状态、幂等**的，调度器不负责管理执行状态。
- 执行时间不宜过长，否则下次触发时可能与前一次重叠。通过 `retry` 和 `timeout` 控制失败行为。
- 适用于数据同步、报表生成、定期清理等批处理场景。

### 配置 heartbeat 服务

heartbeat 在 OpenClaw 里更适合作为“带间隔的守护循环”。它可以被声明为一个长运行任务，由框架负责保持间隔触发。典型配置：

```yaml
tasks:
  - name: redis-stream-consumer
    type: heartbeat
    interval: 5s
    action: plugin://redis-stream/consume
    options:
      singleton: true
      initial_delay: 1s
      launch_policy: at_most_once
```

- `singleton: true` 保证在分布式 Deployment 中只有一个实例持有该 heartbeat，防止重复消费。
- `launch_policy: at_most_once` 配合间隔，如果上一次执行还没结束，下一次触发会被跳过（避免堆叠）。
- `initial_delay` 让任务在启动后稍作等待，避免和依赖服务的就绪检查竞争。

heartbeat 的典型场景：
- 消息队列消费（Redis Stream、Kafka 手动 commit）
- 文件系统变化轮询（非 inotify 环境）
- 外部 API 健康检查并驱动状态机

## 踩坑点

### 1. cron 的时区静默使用 UTC

OpenClaw 早期版本忽略 `timezone` 字段，导致很多北美时区设置的 `0 2 * * *` 在国内服务器上并没有按时触发。排查时，可以直接看调度日志中 `nextRun` 的时间戳，或临时加一个输出当前时间的测试任务来确认。

**修法**：始终在 cron 任务里显式填写 `options.timezone`，不要依赖系统时区。

### 2. heartbeat 执行时间 > 间隔导致“雪崩”

如果业务动作执行耗时 12 秒，而 `interval` 设置为 10 秒，默认行为可能导致任务并发堆积，占用线程池，最终拖垮整个 OpenClaw 运行时。

**修法**：
- 使用 `launch_policy: at_most_once` 或 `skip_if_overlap` 等策略跳过并发。
- 对长时间任务，加大间隔或改造成 cron + 状态机拆分。

### 3. 多实例重复执行 heartbeat

没有设置 `singleton: true` 的 heartbeat 会在每个 OpenClaw 实例上并行运行。如果消费 Redis Stream 的消费者组未正确配置，可能每条消息被不同实例重复处理。

**修法**：
- 设置 `singleton: true` + 后端存储锁（通常基于 etcd 或数据库 advisory lock）。
- 在分布式环境中，保证 heartbeat 任务自身通过框架选举只在一个 leader 上执行。

### 4. cron 任务幂等性不足导致数据错误

cron 失败后 `retry` 再次执行，如果上次部分写入成功，这次又执行，可能出现重复记录。

**修法**：
- 数据写入使用唯一键合并或先标记后清除模式。
- 利用数据库事务或幂等 Key 保证可重入。

## 可复用建议

- **选择 cron 的场景**：定时性明确、频率不高（分钟级以上）、业务逻辑短小且无状态。例如：每小时聚合指标、每天发送摘要报告。
- **选择 heartbeat 的场景**：需要持续监听、轮询外部资源、维护长连接心跳、消费非推送型消息源。例如：Redis 队列消费、WebSocket 保活、目录扫描。
- **混合使用**：可以用 heartbeat 探测工作是否就绪，再触发 cron 任务执行；或 cron 定时重建长时间挂掉的 heartbeat 任务。
- **工程化约束**：
  1. 所有 cron 必须带 `timezone`。
  2. 所有生产级 heartbeat 必须配置 `singleton` 和 `launch_policy`。
  3. 为任务设置合理的 `timeout` 和 `retry`，避免资源泄露。
  4. 为 cron 和 heartbeat 输出结构化日志，区分触发原因和任务 ID，便于追踪。

## 总结

cron 与 heartbeat 不是非此即彼的替代关系，而是在不同时间语义下的互补工具。cron 解决“在特定时刻做某事”，heartbeat 解决“每隔一段时间检查/执行某事”。正确使用两者，可以让 OpenClaw 上的自动化流程既准时又可靠，避免因为滥用 cron 导致实时性差，或者因为 heartbeat 缺乏约束而导致资源耗尽。

下次在配置文件中写下 `schedule` 或 `interval` 时，不妨多问一句：这个任务是“定时的批处理”还是“持续的守护循环”？答案会自然导向合适的类型。

---

