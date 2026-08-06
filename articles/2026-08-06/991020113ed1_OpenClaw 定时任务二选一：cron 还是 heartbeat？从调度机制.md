---
title: OpenClaw 定时任务二选一：cron 还是 heartbeat？从调度机制到避坑指南
feedId: 31887
source: 综合讨论
publishedAt: 2026-08-06
---

## 背景：两种“定时”，完全不同的两套逻辑

在 OpenClaw 的 Agent 编排体系里，你会接触到两种周期性任务机制：

- **全局 cron**：在 `plugins` 配置中通过类似 CronTrigger 的方式声明，由框架的调度线程池统一管理；
- **Agent heartbeat**：每个 Agent 实例内建的 `on_heartbeat` 生命周期回调，按固定间隔在 Agent 自己的上下文里触发。

很多实践者一开始会把它们都当成“定时器”来用，结果发现：
- cron 任务没按预期时间触发；
- heartbeat 偷跑、堆积，甚至 Agent 重启后任务静默消失。

根源在于：**它们面向的调度边界、可靠性和扩展成本完全不同**。下面用一次真实的缓存清理需求，把两条路都走一遍。

---

## 问题：同一个需求，两条腿走路

场景：你有多个 Agent 实例，在 OpenClaw 集群模式下运行。每个实例本地维护一份临时文件缓存，需要每 5 分钟清理一次超过 1 小时的陈旧文件。

方案 A：写一个全局 cron 表达式 `0 */5 * * * *`，让它调用某个内置 tool 去扫描并删除。  
方案 B：在每个 Agent 的 `on_heartbeat` 里实现清理逻辑，interval 设为 300s。

两条路都能跑起来，但生产环境下的表现天差地别。

---

## 实践步骤

### 1. Cron 方案的配置与实现

在 `openclaw.yaml` 的 plugin 定义中绑定一个 cron trigger：

```yaml
plugins:
  - name: cache-cleaner
    type: builtin
    triggers:
      - type: cron
        expr: "0 */5 * * * *"
    actions:
      - tool: clean_temp_cache
        params:
          ttl_seconds: 3600
```

`clean_temp_cache` 工具内实现幂等删除逻辑（例如基于文件修改时间，加 `find -mmin +60 -delete`）。  
> 踩坑1：cron 表达式底层基于框架的调度器，**时区通常为 UTC**，与本地时间存在偏差。遇到准点不触发，先检查 `date -u` 对应的实际分钟时间。

### 2. Heartbeat 方案的实现

在 Agent 代码中重写 `on_heartbeat`：

```python
class CacheAwareAgent(BaseAgent):
    async def on_heartbeat(self):
        # 防止前一次清理还在运行导致堆积
        if self._cleaning:
            return
        self._cleaning = True
        try:
            await self.clean_cache(ttl=3600)
        finally:
            self._cleaning = False
```

并在 Agent 初始化时注册 heartbeat 间隔：

```python
agent = CacheAwareAgent(
    name="cleaner",
    heartbeat_interval=300  # 秒
)
```

> 踩坑2：heartbeat 间隔 ≠ 执行间隔。如果 `clean_cache` 耗时 10 秒，第 2 次心跳完全可能在 290 秒后就进来，因为框架在任务完成后会重新计算下一个定时。**简单加锁只是防并发，不能防堆积**。需要额外带上“最后执行时间戳”有条件判断（如间隔小于 280 秒就跳过）。

---

## 踩坑盘点与根因分析

### Cron 方案的隐患

- **多实例重入**：集群有 3 个节点，每个节点都跑同一个 cron plugin，会同时触发三次清理。**必须用分布式锁**（如 Redis SETNX），否则会造成 IO 风暴或文件误删。
- **调度线程池耗尽**：所有 cron 任务共用一个固定大小的线程池。如果某个任务阻塞（比如等锁超时），会拖慢其他 cron 执行，表现为“整点错位”。
- **冷启动不补偿**：调度器错过的时间段不会补偿执行。这一点和 Linux crond 不同，重启后只会从当前时间往后匹配。

### Heartbeat 方案的细坑

- **与 Agent 生命周期强绑定**：Agent 一旦 crash，heartbeat 会彻底消失，且不会发出任何告警。对于清理类任务，可能缓存放飞好几天才发现。
- **执行时间叠加导致雪崩**：这是最危险的模式。假设 heartbeat 间隔 60s，正常清理耗时 3s；某天积压大量文件后耗时飙升到 80s——下一个心跳到达时检测到未完成，跳过；但标志位状态可能因异常未正确重置，导致后续心跳全部跳过，任务永久停滞。
- **无集中监控**：heartbeat 的触发散落在每个 Agent 进程里，不像 cron 有统一的 trace，排障成本高。

---

## 可复用的决策与建议

别把 heartbeat 当成“不要钱的小 cron”。下面这张简表可以贴在你的设计文档里：

| 维度 | Cron 调度器 | Agent Heartbeat |
|------|------------|-----------------|
| 触发边界 | 全局（所有 node） | 单 Agent 实例内 |
| 可靠性 | 随框架进程存活，需自己做锁 | 随 Agent 存活，Agent 死则任务死 |
| 适合任务 | 全局唯一、低频、关键路径（数据同步、报表推送） | 轻量、与本 Agent 状态紧耦合的探活/自愈 |
| 最大风险 | 多实例重复执行、线程池饥饿 | 静默停止、任务堆积雪崩 |
| 运维要求 | 集中监控、分布式锁 | 心跳保活监控、最大耗时上限保护 |

**工程化实践建议**：

1. **cron 任务一律实现幂等 + 分布式锁**，并在任务入口打结构化日志（start/end/result）。
2. **heartbeat 里绝不写 I/O 耗时操作**；如必须，放入异步队列或协程，设定 2×interval 的上限超时，超时即告警。
3. **混合使用**：用 cron 做“大脑”级的编排（例如每 10 分钟检测所有 Agent 的健康状态）;用 heartbeat 做“局部”的反应式维护（单个 Agent 内部的连接池清理、内存重载检查）。
4. 给 heartbeat 加上 **liveness gauge**：每次成功心跳，向监控系统发送一次计数器；面板上若 5 分钟内没有增量，立即通知。

---

## 总结

OpenClaw 的 cron 和 heartbeat 并不对立，但它们解决的是不同粒度的周期性问题。把 heartbeat 当成“免费 cron”用，是最大的设计失误。真正工程化的选择是：**全局的任务走 cron，并做好集群下的去重与锁；实例级的自维护走 heartbeat，但必须守卫最大执行时间，并建立心跳保活监控。** 两者组合，才能让你的 OpenClaw 编排既稳定又可观测。

---

