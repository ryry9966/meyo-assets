---
title: OpenClaw 定时任务防重入锁：为什么 5 分钟任务会撞车
feedId: 29702
source: 综合讨论
publishedAt: 2026-07-20
---

## 背景

在 OpenClaw 里通过 MCP 插件或内置 Scheduler 定义定时任务，是很多自动化 Agent 落地的第一步。典型场景：每 5 分钟让 Agent 去拉取一次外部数据、生成摘要再写入 Notion。调度稳定时一切正常，直到某天你发现数据库里出现了大量重复记录，翻日志才看到同一个任务竟然有两个实例在同时跑——这就是“撞车”。

```
[12:00] 触发第一次执行
[12:05] 触发第二次执行，但第一次还没结束
 → 两实例并行操作同一份资源
```

根本原因很简单：**没有防重入机制**。在 Agent 编排中，这类问题比传统后端更隐蔽，因为 Agent 的耗时波动很大（模型推理、API 速率限制、人工审批等），5 分钟的 cron 很容易被一个偶尔跑 6 分钟的任务打穿。下面给出在 OpenClaw 环境下可工程化落地的解决方案。

## 问题分析

OpenClaw 的定时触发器默认只保证“按时启动”，不检查上一次是否完成。核心冲突来自：

- 触发间隔 < 任务最大可能执行时间
- 任务内部有状态写入（数据库、文件、Redis）
- 没有外部协调机制

对于纯无状态只读任务，撞车只是浪费算力；一旦涉及写入，就会产生数据重复、乐观锁冲突、甚至脏数据。防重入本质上是在**进程级别加互斥锁**，确保同一个任务逻辑同一时刻只有一个实例在跑。

## 做法 / 步骤

以 OpenClaw 的 MCP Python 插件为例，我们引入一个基于 Redis 的轻量级分布式锁，不依赖外部调度框架。

### 1. 锁工具实现

```python
import redis
import uuid
import time

r = redis.Redis(host='localhost', port=6379, db=0, decode_responses=True)

def acquire_lock(lock_key: str, ttl: int = 60) -> str | None:
    """尝试加锁，返回锁标识符（token），失败返回 None"""
    token = str(uuid.uuid4())
    # SET NX EX 原子操作
    if r.set(lock_key, token, nx=True, ex=ttl):
        return token
    return None

def release_lock(lock_key: str, token: str):
    """安全释放：只有持有者才能释放"""
    script = """
    if redis.call("GET", KEYS[1]) == ARGV[1] then
        return redis.call("DEL", KEYS[1])
    else
        return 0
    end
    """
    r.eval（script, 1, lock_key, token)
```

### 2. 在 MCP 工具函数上加锁

假设你的 MCP 工具是 `fetch_and_sync`：

```python
from functools import wraps

def with_task_lock(lock_key: str, ttl: int = 600):
    def decorator(func):
        @wraps(func)
        async def wrapper(*args, **kwargs):
            token = acquire_lock(lock_key, ttl)
            if token is None:
                print("任务已在执行，跳过本次")
                return {"status": "skipped"}
            try:
                return await func(*args, **kwargs)
            finally:
                release_lock(lock_key, token)
        return wrapper
    return decorator

@with_task_lock("cron:fetch_sync", ttl=600)
async def fetch_and_sync(source: str):
    # 实际业务逻辑
    ...
```

`lock_key` 按任务全局唯一，`ttl` 必须**大于任务最长可能执行时间**。我这里设了 10 分钟，给 5 分钟任务留足余量。

### 3. 在 OpenClaw 中注册

MCP 服务器通过 `manifest.json` 注册后，/invoke 调用会自动走装饰器逻辑。无需改动 Scheduler 配置。

## 踩坑点

### 1. TTL 太小，锁提前释放
最常见的翻车。比如任务正常 4 分钟，但某次因 API 无限重试跑了 6 分钟，锁在 5 分钟时自动过期，第二个实例抢到锁开始写数据，撞车照样发生。**TTL 务必按上限设置**，不要为了怕死锁而设得太短。

### 2. 直接 DEL 释放锁
如果不判断持有者直接 `r.delete(lock_key)`，会导致 A 实例任务超时释放后，B 实例获得锁，然后 A 执行完成时误删了 B 的锁，系统状态错乱。必须用 Lua 脚本原子校验。

### 3. 进程崩溃导致死锁
如果 `finally` 没能执行（比如进程被 kill -9），锁会残留在 Redis 中直到 TTL 过期。这就是为什么 TTL 不能无限大的原因。**可以加一个 watch dog 线程定时续期**，但如果在 OpenClaw 托管环境里不方便，建议将 TTL 设为任务合理上限加 20% buffer，并做好告警。

### 4. Redis 不可用
锁依赖 Redis，Redis 故障会导致所有任务跳过或全部失败。要根据业务选择策略：宁可错过一次执行（降级）还是冒重复风险。可以在 `acquire_lock` 失败时记录 metric 并报警，不要静默吞掉。

## 可复用建议

1. **任务粒度**：一个 cron 对应一个 lock_key，不要在同一个锁里串行多个不相关的任务。
2. **锁超时公式**：`TTL = 历史 P99 耗时 × 1.5 + 额外缓冲`，而不是固定值。
3. **可观测性**：记录获锁成功/失败次数、锁竞争次数，方便评估是否该延长触发间隔或优化任务效率。
4. **备选方案**：数据库唯一索引做任务执行记录（如 `insert task_execution(task_id, start_time)` 并检查是否已有未结束记录），适合无 Redis 的场景，但要注意清理历史记录。
5. **与 OpenClaw 集成**：若未来平台原生支持防重入，建议迁移；在此之前，这套装饰器方案不侵入业务代码，方便拆除。

## 总结

“5 分钟任务撞车”看似低级，但在 Agent 自动化里非常典型，因为 LLM 响应的不确定性放大了传统调度的小概率边界条件。核心解法就是一个可靠的分布式锁，但细节（TTL 设定、安全释放、故障兜底）直接决定生产可用性。在 OpenClaw 的 MCP 插件体系下，用 Redis + Lua 可以低成本实现，让你的定时 Agent 真正变成“火力单元”，而不是定时炸弹。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-20/05cc84e9e65fc42c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-20/c98e198c877bce40.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-20/dba60fec1555ff10.png)

