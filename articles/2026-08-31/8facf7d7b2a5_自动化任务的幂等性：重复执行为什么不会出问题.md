---
title: 自动化任务的幂等性：重复执行为什么不会出问题
feedId: 35460
source: 综合讨论
publishedAt: 2026-08-31
---

## 背景

在 OpenClaw、Agent、MCP 以及各类插件自动化的实践中，任务通常由定时器、Webhook 回调、消息事件或模型决策触发。一次真实的业务动作，可能在以下场景中被重复触发：

- 定时任务因网络抖动触发重试；
- 消息队列在消费超时后重新投递；
- Agent 重复调用同一个工具；
- 多实例部署时没有全局锁；
- 用户在界面上多次点击触发。

如果处理函数不具备幂等性，第二次执行会再次产生副作用，例如重复写入数据、重复发送消息、重复调用外部付费 API，甚至造成数据状态被覆盖。

## 问题

幂等性（Idempotency）指的是：同一个操作执行一次或多次，最终结果一致，不会产生额外副作用。

我在接入飞书机器人定时推送时踩过一个典型坑：每天 09:00 拉取当天日程并推送卡片。某次因请求超时，调度器自动重试了一次，结果用户收到两张完全相同的卡片。另一个场景是 Agent 调用创建工单插件，模型在一次推理中重复调用了工具，结果生成了重复工单。

这些问题的共同点不是“重试”本身，而是任务没有根据业务唯一标识做去重。

## 做法与步骤

在 OpenClaw 插件或自动化任务脚本中，我通常用以下模式实现幂等：

### 1. 定义稳定的业务幂等键

每个任务在开始前确定一个唯一标识。例如：

- 定时推送：`daily_schedule_{yyyy-MM-dd}`
- 创建工单：`create_ticket_{request_id}`
- 文件同步：`sync_file_{file_hash}`

键中不要包含时间戳、随机数、日志 ID 等非确定性参数。

### 2. 持久化执行记录

使用 SQLite、PostgreSQL、Redis 或本地文件保存执行状态。典型表结构：

```sql
CREATE TABLE task_executions (
    task_key   TEXT PRIMARY KEY,
    status     TEXT NOT NULL,          -- running / success / failed
    result     TEXT,
    created_at INTEGER,
    updated_at INTEGER
);
```

核心是 `task_key` 的唯一约束。

### 3. 先占位再执行

任务入口不要直接执行业务逻辑，而是先尝试插入 `running` 状态：

```sql
INSERT INTO task_executions(task_key, status)
VALUES(?, 'running')
ON CONFLICT(task_key) DO NOTHING;
```

如果插入成功，说明拿到了执行权；如果插入失败，说明该任务已经在处理中或已完成，直接读取已有结果返回，或者丢弃本次触发。

使用 Redis 时可以用 `SET task_key 1 NX PX 3600` 做轻量锁，但注意锁的超时时间要大于业务最长执行时间，否则可能提前释放导致并发。

### 4. 更新最终状态

业务执行成功后更新为 `success`，执行失败更新为 `failed` 并保留错误信息。下一个相同 key 的触发到来时，直接从记录中返回历史结果，不会重复执行。

### 5. 使用天然幂等的写操作

对于数据库写入，优先使用 `INSERT ... ON CONFLICT DO UPDATE` 或 `INSERT IGNORE`，避免先查后写产生竞态。对外部 HTTP API，尽量使用 PUT 而不是 POST，或在请求头中带上 `Idempotency-Key`。

## 踩坑点

- **只在成功时记录**：如果任务执行到一半崩溃，没有留下 `running` 或 `failed` 记录，下次重试会再次执行。必须先插入占位，即使失败也有状态可查。
- **幂等键粒度不当**：键太宽会导致漏执行，例如按天聚合但业务需要处理每次事件；键太窄则失去去重效果，例如包含随机数。
- **并发窗口未关闭**：先查后写在多线程或多实例下会同时通过检查。必须依赖数据库唯一约束或 Redis `NX` 原子操作，不能只靠应用层判断。
- **键包含非确定性参数**：例如 `now()`、随机字符串会让每次 key 都不同，幂等设计形同虚设。
- **任务内部多次外部调用**：即使任务整体幂等，但若在调用外部邮件服务成功后进程崩溃，重试后可能重复发送。此时需要把外部调用拆成可独立重放的步骤，或为外部调用单独做幂等记录。

## 可复用建议

在 OpenClaw 插件中，我会封装一个通用装饰器：

```python
def idempotent(key_func):
    def decorator(fn):
        def wrapper(*args, **kwargs):
            key = key_func(*args, **kwargs)
            db = get_connection()
            inserted = db.execute(
                "INSERT INTO task_executions(task_key, status) VALUES(?, 'running') "
                "ON CONFLICT(task_key) DO NOTHING",
                (key,)
            )
            if inserted == 0:
                row = db.query("SELECT result FROM task_executions WHERE task_key=?", (key,))
                return row['result']
            try:
                result = fn(*args, **kwargs)
                db.execute(
                    "UPDATE task_executions SET status='success', result=?, updated_at=? WHERE task_key=?",
                    (result, now(), key)
                )
                return result
            except Exception as e:
                db.execute(
                    "UPDATE task_executions SET status='failed', result=?, updated_at=? WHERE task_key=?",
                    (str(e), now(), key)
                )
                raise
        return wrapper
    return decorator
```

使用时：

```python
@idempotent(lambda date, channel: f"daily_brief_{date}_{channel}")
def run_daily_brief(date, channel):
    ...
```

对于 Agent 工具调用，可以在工具输入 schema 中显式暴露 `idempotency_key` 字段，让上游调用方传入稳定的请求 ID。

## 总结

幂等性不需要复杂的框架，在 OpenClaw 体系中用数据库唯一约束加状态占位就能解决大部分重复执行问题。核心就四点：**稳定 key、持久化记录、原子占位、状态更新**。

把幂等设计纳入自动化任务的第一步，可以避免后续大量人工清洗数据和用户投诉。重复执行不应该靠运气避免，而是从机制上保证“执行两次等于执行一次”。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/c22ee79e3c76b762.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/793c89e8195e824e.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/0f4ddff021caafed.png)

