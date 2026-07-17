---
title: OpenClaw 定时任务防重入锁：为什么 5 分钟任务会撞车
feedId: 29365
source: 综合讨论
publishedAt: 2026-07-17
---

## 背景

OpenClaw 生态中，很多自动化能力靠定时任务驱动：数据同步、缓存清理、过期会话回收、Agent 状态轮询。平台内置了 cron 调度器，开发者只需要注册一个 handler，就能以分钟级甚至秒级精度定时触发逻辑。

一切正常，直到你发现一个配置为每 5 分钟执行一次的任务，偶尔会出现两个实例同时跑。日志里开始出现奇怪的竞争：数据库记录被重复更新，文件被两次删除导致报错，甚至 Agent 的输出互相覆盖。原因不是 cron 调度错了，而是**任务执行时间偶尔超过间隔**，下一轮触发时上一轮还没结束，两个实例撞了车。

本质是 OpenClaw 的调度器**不检查前一任务是否完成**，只是按计划触发。如果你的 handler 内部没有任何同步保护，就会产生并发执行。这就是典型的“定时任务重入”问题，在工程中并不新鲜，但在 Agent 流程中特别致命——因为 Agent 调用 LLM、等待工具返回的时间波动很大，5 分钟的任务跑到 8 分钟并不罕见。

## 问题解剖

假设我们有一个每 5 分钟触发一次的 `cleanup-expired-sessions` 任务，伪代码：

```ts
export async function handler(ctx: CronContext) {
  const sessions = await ctx.db.findExpired();
  for (const s of sessions) {
    await ctx.db.deleteSession(s.id);
    await ctx.fileStorage.remove(s.uploadDir);
  }
}
```

正常时 2 分钟跑完，一切太平。某天过期会话突然增多，或者文件存储在慢速 NAS 上，这次执行花了 7 分钟。在它启动后的第 5 分钟，调度器又触发了新的 `handler` 实例。两个实例同时扫描过期会话、尝试删除同一记录和文件，轻则报“记录不存在”“文件不存在”，重则删除操作互相干扰，留下残留目录或锁表。

在 Agent 编排场景中，这种情况更隐蔽：一个定时 Agent 可能在等待工具调用、模型推理时沉睡数分钟，调度器却在不知情的情况下再次启动它，导致同一任务链运行两遍，状态机进入非法状态。

## 防重入的做法：本地文件锁方案

OpenClaw 的运行时通常是 Node.js 进程，支持多 worker。在单实例部署（最常见）下，最简单的防重入机制是使用**操作系统级别的排他文件锁**。Node 的 `fs.open` 配合 `'wx'` 标志可以实现原子性的锁文件创建，适合轻量任务。

### 步骤

1. **定义锁文件路径**  
   使用任务名作为锁标识，将锁文件放在可写的临时目录，例如 `/tmp/openclaw-locks/cleanup-expired-sessions.lock`。确保目录存在。

2. **封装锁获取函数**

   ```ts
   import fs from 'node:fs';
   import path from 'node:path';

   const LOCK_DIR = '/tmp/openclaw-locks';

   async function acquireLock(taskName: string, ttlMs: number = 10 * 60 * 1000): Promise<() => Promise<void>> {
     const lockPath = path.join(LOCK_DIR, `${taskName}.lock`);
     await fs.promises.mkdir(LOCK_DIR, { recursive: true });
     const fd = await fs.promises.open(lockPath, 'wx');
     const release = async () => {
       await fd.close();
       await fs.promises.unlink(lockPath).catch(() => {});
     };
     // 写过期时间，用于后续清理死锁（可选）
     await fd.write(String(Date.now() + ttlMs));
     return release;
   }
   ```

   关键点：`'wx'` 标志保证只有在文件不存在时才能创建，这是原子操作，避免了“检查再创建”的竞态。如果文件已存在，`open` 会直接抛出 `EEXIST` 异常。

3. **在任务 handler 中使用**

   ```ts
   export async function handler(ctx: CronContext) {
     let release: (() => Promise<void>) | null = null;
     try {
       release = await acquireLock('cleanup-expired-sessions', 10 * 60 * 1000);
     } catch (err: any) {
       if (err.code === 'EEXIST') {
         ctx.logger.warn('Previous instance still running, skipping this run');
         return;
       }
       throw err;
     }
     try {
       // 原有业务逻辑
     } finally {
       await release();
     }
   }
   ```

4. **处理死锁**  
   如果任务抛异常导致 `release()` 未被调用，锁文件会残留，后续任务永远拿不到锁。解决方法是在 `acquireLock` 中增加锁过期检查：当获取失败时，读取锁文件记录的时间戳，如果已超过 TTL，就认为上一实例已经死亡，删除锁文件并重试。注意这个检查与删除之间仍有竞态，但对于非高频任务，用一个简单的重新尝试循环已经足够健壮。更严谨的方案可以使用 `fs.open` 加 `flock` 系统调用，但跨平台性略差。

## 踩坑点

- **锁文件目录权限**  
  OpenClaw 运行用户必须对 `/tmp/openclaw-locks` 有读写权限。在容器化部署中，注意不要使用 root 跑应用，应预先创建目录并 chown。

- **异常退出遗留**  
  Node 进程被 SIGKILL、OOM 杀死时，`finally` 不执行，锁文件永久残留。因此必须设计过期清理。一种低成本方案：每次成功获取锁后，启动一个定时器，在自己未释放锁的前提下延长过期时间；或者依靠外部 cron 每分钟清理一次过期锁文件。

- **锁粒度**  
  任务名必须唯一，如果两个不同项目复用同一 lock 目录，注意加前缀隔离（如 `projectA:cleanup`）。

- **多实例 / 多进程部署**  
  文件锁只在同一台机器上有效。如果一个 OpenClaw 服务启了多个 worker 进程，它们共享文件系统，文件锁依然有效。但如果是容器化且多个 Pod 挂载不同临时目录，或者使用了 distroless 等无磁盘环境，文件锁就不适用了。此时应切换到外部分布式锁（Redis `SET NX` 或数据库行锁）。

- **TTL 设置**  
  锁过期时间必须大于任务的最长可能执行时间。Node 事件循环阻塞会延长实际耗时，保守设置 2-3 倍最长时间。太短的 TTL 会导致任务未完成锁自动释放，另一个实例长驱直入，重入问题再现。

## 复用建议

不要在每个任务里散落锁逻辑，应抽成通用 wrapper：

```ts
export function withLock(taskName: string, ttlMs: number, fn: (ctx: CronContext) => Promise<void>) {
  return async (ctx: CronContext) => {
    // 锁获取、过期处理、释放
    // 调用 fn(ctx)
  };
}
```

然后注册任务时只需：

```ts
cronService.register('cleanup-expired-sessions', '*/5 * * * *', withLock('cleanup', 600_000, handler));
```

同时建议：

- 对锁获取失败、锁过期释放等关键事件输出结构化日志，设置告警。
- 如果业务允许，可进一步做任务幂等设计——即使偶尔重入，也不产生副作用（例如依靠数据库唯一键、乐观锁）。
- 在 OpenClaw 管理界面或运维看板中暴露任务是否“被锁”的状态，方便排查。

## 总结

OpenClaw 的定时调度器是朴素的触发器，它不关心任务是否还在跑。5 分钟间隔的任务，只要有一次跑超了，就会形成并发。解决思路本质上就是给任务加一个互斥锁，确保同一时刻只有一个实例在运行。

文件锁方案简单、无外部依赖，适合单机部署的大多数场景。多实例部署时替换为 Redis 分布式锁，逻辑几乎不变。关键是要覆盖异常退出导致死锁的情况，用过期机制兜底。

当你的 Agent 定时链开始出现诡异的双重输出、数据冲突时，先别怀疑模型幻觉，检查一下是不是两个副本在互相踩脚。

---

