---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 33241
source: 综合讨论
publishedAt: 2026-08-15
---

## 背景

在 OpenClaw 里跑 Agent、MCP 或插件时，最让人不安的不是“它做错了”，而是“它用一个 `rm -rf` 把宿主目录带走了”。很多自动化事故不是模型能力问题，而是执行边界没收紧：脚本要写临时文件、插件要访问缓存、MCP 工具要读项目目录，最后所有操作都落在同一个进程权限里。

OpenClaw 的 sandbox 解决的不是“让 Agent 变聪明”，而是让危险操作没有落点。它的核心思路是：**默认只读、显式可写、删除重定向、可回滚**。

## 问题

一个典型场景：你让 Agent 整理某个目录，它先扫描文件，再调用 shell 删除它认为的重复项。如果 shell 直接跑在宿主机，删除就是真删除。即使模型没有恶意，路径拼错、变量为空、通配符扩散，都可能造成误删。

另一个场景来自 MCP/插件。MCP 工具通常是外部进程，权限模型不一定和你主 Agent 一致。如果它拿到宿主机路径，又没有被限制在沙箱里，它的“清理缓存”可能清掉项目文件。

所以需要回答：**怎样让 OpenClaw 的 Agent 在访问文件时，具备隔离和回滚能力，而不是依赖提示词说“请小心”。**

## 做法/步骤

### 1. 让所有任务在 workspace 内启动

OpenClaw 的任务进程应启动在独立 workspace，例如 `/var/tmp/openclaw/workspaces/{run_id}`。不要在宿主 `~/` 下直接执行任务。这样最坏情况下，破坏范围也被限制在 run 目录。

### 2. 设置只读挂载

需要读取的宿主目录，例如 `~/projects`，以只读方式挂载到沙箱内：

```yaml
sandbox:
  mounts:
    - src: ~/projects
      dst: /mnt/projects
      mode: ro
```

Agent 能看到项目文件，但无法删除或覆盖 `/mnt/projects` 下的内容。删除尝试会被文件系统层拒绝，而不是等 rm 执行完再后悔。

### 3. 可写路径使用安全写

如果 Agent 需要输出结果，应指向明确的可写目录，并开启 safe-write 机制。示例：

```yaml
sandbox:
  mounts:
    - src: ~/out
      dst: /mnt/out
      mode: rw
      safeWrite: true
```

safe-write 的核心不是拦截写，而是避免直接覆盖原文件。新的写入会走临时文件 + 原子替换，减少任务中途失败导致文件损坏。

### 4. 删除操作重定向到回收站

这是“不会误删文件”最关键的一步。OpenClaw 的 sandbox 支持将删除操作从真删改成移动到回收站：

```yaml
sandbox:
  deletion:
    mode: trash
    trashPath: /var/tmp/openclaw/trash/{run_id}
    autoExpire: 7d
```

这样 Agent 执行 `rm` 时，文件并没有从磁盘消失，而是被移入 run 级别回收站。任务结束后可以统一检查、恢复或定期清理。

### 5. 给 MCP/插件单独的权限 profile

不要给 MCP 工具和主 Agent 完全相同的文件权限。对 MCP 可以只给一个最小的可访问路径集合，并禁用直接 shell 执行：

```yaml
mcp:
  profiles:
    default:
      allowShell: false
      allowedPaths:
        - /mnt/projects
        - /mnt/out
```

即便某个 MCP 工具行为异常，它也没有宿主机 shell 可用，且文件访问范围被收紧。

## 验证方法

写一个最小用例验证边界是否生效：

```bash
mkdir -p ~/sandbox_test
echo "important" > ~/sandbox_test/keep.txt

# 让 Agent 在任务中执行：
rm /mnt/projects/keep.txt
```

检查结果应该是：删除被拒绝，或者文件出现在回收站，而 `~/sandbox_test/keep.txt` 原文件仍在。不要只看 Agent 返回“已删除”，要核对宿主目录。

## 踩坑点

1. **只做 chroot 不够。** 如果可写目录和只读目录被混挂到同一棵树下，Agent 可能通过 `..`、软链接或 `/proc/self/fd` 绕过限制。尽量让只读和可写路径分离到 `/mnt` 下不同挂载点。
2. **回收站不等于磁盘空间释放。** `trash` 模式下，文件仍占用磁盘。高频率删除任务会让 `/var/tmp` 膨胀，必须配置 `autoExpire` 或定期清理。
3. **不要给插件 `CAP_DAC_OVERRIDE` 或 root 等价能力。** 这类 capability 会绕过普通文件权限检查，让只读挂载形同虚设。
4. **路径穿越测试要持续做。** 每次调整挂载配置后，跑一遍 `rm /mnt/ro/../ro/file`、`rm` 软链接目标等回归用例。
5. **配置示例字段可能随版本变化。** 不同版本的 OpenClaw 沙箱参数名称不完全一致，上线前以当前版本文档和 `sandbox` 配置校验结果为准。

## 可复用建议

- **默认只读，显式可写。** 能不改宿主的目录，一律 `ro` 挂载。
- **删除优先进回收站，而不是直接拒绝。** 直接拒绝会让 Agent 反复重试；回收站既保留了安全边界，又保持任务可继续执行。
- **把 MCP 和主 Agent 的权限分开。** 插件生态越丰富，越要限制进程级权限。
- **给任务加自动快照。** 如果任务涉及批量改写或迁移，开启 run 前快照，失败时回滚。
- **审计拒绝日志。** sandbox 的删除拒绝、写拒绝记录是很好的安全信号，不要关掉。

## 总结

OpenClaw 的 sandbox 并不是一个“更聪明的 rm”，而是一套分层文件访问边界：**隔离 workspace、只读挂载宿主目录、可写输出走安全写、删除重定向到回收站、MCP 独立受限**。这比单纯在提示词里写“不要误删文件”可靠得多。

工程上，安全边界要能被测试和回滚。如果你正在 OpenClaw 上跑自动化任务，先把这五步配置验证一遍，再考虑让 Agent 接触更敏感的真实目录。

---

