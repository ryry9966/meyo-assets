---
title: OpenClaw Sandbox 防误删机制拆解：默认拒绝、可撤销删除与审计
feedId: 34314
source: 综合讨论
publishedAt: 2026-08-23
---

## 背景

很多 OpenClaw 用户第一次把 Agent 接进真实目录时，最担心的不是任务失败，而是 Agent “抽风”执行了类似 `rm -rf ~/` 或误删项目文件。社区里也常出现“文件不见了”的反馈。

实际上，OpenClaw 的 sandbox 安全模型并不是简单地把 Agent 关进一个目录，而是通过 **默认拒绝 + 可撤销删除 + 审计日志** 组成多层防护。理解这层模型，比单纯备份更重要：它决定了 Agent 在什么情况下能碰文件，什么情况下只能“看起来能碰”。

## 问题：为什么还会误删？

工程上，OpenClaw 的 sandbox 不能保证 100% 不删文件，它保证的是：

1. Agent 默认只能写 workspace，不能直接写宿主目录；
2. 删除动作默认进入 trash/review，而非直接 unlink；
3. 所有高风险操作有日志可查。

真正导致“文件没了”的，通常是以下配置错误或绕过路径：

- 把家目录直接挂载为可写；
- 把删除策略改成 `immediate` 或等价选项；
- 以 root 运行 sandbox，导致部分文件权限失效；
- 安装的 MCP/插件直接暴露本地文件 API，绕过了 sandbox；
- workspace 内存在符号链接，删除时跟随到外部目标。

所以，OpenClaw 的 sandbox 不是“魔法安全”，而是一套需要正确配置和理解的边界。

## 做法/步骤：四步确认沙箱边界

以下步骤以社区常见配置为例，具体字段名以你当前 OpenClaw 版本为准。

### 1. 确认可写根目录

只给任务目录写权限，不要让 Agent 直接写家目录或系统目录。

```
sandbox:
  mode: workspace-only
  root: /home/user/openclaw/workspace
```

`mode` 应避免使用 `host`、`unsafe` 或 `full-access` 之类放开宿主文件系统的选项。如果你需要 Agent 读取某个外部目录，用只读挂载，而不是直接给可写权限。

### 2. 删除策略改为 trash 或 review

默认情况下，Agent 的删除操作不应直接进入 `unlink`，而是进入回收目录或等待审核。

```
sandbox:
  delete: trash
  trash_keep_days: 7
```

这样即使 Agent 误删，文件也只是被移动到了 sandbox 的回收区，7 天内可以恢复。对于高风险任务，可以进一步设为 `review`，删除动作会先挂起，由人工确认。

### 3. 限制符号链接跟随

符号链接是沙箱逃逸的常见入口。workspace 内如果有一个软链指向 `~/project`，Agent 删除软链时若沙箱跟随了链接，可能直接操作外部文件。

```
sandbox:
  no_symlink_follow: true
```

开启后，Agent 对符号链接本身的操作不会影响到目标路径。这也意味着如果你确实需要让 Agent 通过软链访问外部资源，需要显式声明，不要默认开启。

### 4. 用非 root 用户运行

即使 sandbox 内部提供了文件系统隔离，如果进程本身以 root 运行，仍然可能绕过部分权限检查。建议在启动 OpenClaw 时使用普通用户，并对 workspace 目录设置合理的属主和权限。

## 踩坑点

### 1. MCP 插件绕过 sandbox

不少 MCP server 为了“方便”，直接暴露了本机文件读写 API。sandbox 只能约束 OpenClaw 核心的文件操作，无法约束插件自己发起的系统调用。所以只安装可信 MCP，并尽量限制 MCP 的运行权限。

### 2. 回收站占满磁盘

`delete: trash` 看似安全，但如果长期不清理，回收目录可能积累大量文件，导致磁盘空间耗尽。建议设置 `trash_keep_days`，并定期检查回收目录大小。

### 3. 把“不会误删”理解为“完全不能删”

OpenClaw 的 sandbox 允许 Agent 在授权范围内删除文件。如果任务本身需要清理临时文件，Agent 仍然会执行删除，只是这些删除经过 trash 或审核。不要把安全模型当成“Agent 永远不会删除任何东西”。

### 4. 只依赖 sandbox，不做备份

sandbox 只能降低误删概率，不能替代备份。对于重要目录，仍然需要独立备份或版本控制。

## 可复用建议

- **最小权限**：只给任务所需目录写权限，其他目录只读或不可见。
- **删除保护优先**：保持 `trash` 策略，不要为了“少一步确认”改成 `immediate`。
- **开启审计日志**：删除、移动、覆盖等高危操作应记录到日志，方便事后排查。
- **一次性环境**：对于不信任的任务，使用临时容器或 overlay 文件系统，任务结束后直接丢弃。
- **定期演练**：用一个测试任务尝试删除 sandbox 外的文件，验证策略是否生效。把“验证沙箱边界”当成上线前检查项。

## 总结

OpenClaw 的 sandbox 安全模型不是“不允许删除”，而是让删除变得可预测、可撤销、可审计。它默认把 Agent 限制在工作目录，删除操作走 trash 或 review，符号链接不跟随，权限以普通用户运行。只要配置正确，Agent 误删宿主文件的概率会大幅降低；但一旦放开挂载、关闭保护或使用不受控的 MCP，安全边界就会失效。

真正能让你安心的，不是默认配置本身，而是你理解并维护这套边界的方式。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/81d2f6db07241af9.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/1451575432b6ccf6.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/9754ec819fc0453d.png)

