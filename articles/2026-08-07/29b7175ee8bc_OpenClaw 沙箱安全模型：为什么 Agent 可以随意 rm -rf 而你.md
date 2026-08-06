---
title: OpenClaw 沙箱安全模型：为什么 Agent 可以随意 rm -rf 而你的文件毫发无伤
feedId: 31907
source: 综合讨论
publishedAt: 2026-08-07
---

## 背景

让 LLM Agent 直接操作文件系统是一件危险的事。无论是自主任务规划时的“幻觉”，还是用户输入了恶意指令，一句 `rm -rf /` 都可能让服务器当场瘫痪。常见思路是用 Docker 容器隔离，但启动开销大，并且容器内 Agent 仍需挂载宿主目录才能工作，隔离边界存在泄漏风险。

OpenClaw 作为面向自动化实践者的 Agent 框架，采用了一套**进程级轻量沙箱**，不依赖完整容器运行时，却能让 Agent 对文件系统的一切破坏性操作都落在临时层上，而真实数据分毫不损。下面拆解这套机制的原理、配置方法与踩坑点。

## 问题

我们希望 Agent 能自由读写指定目录，但又要确保：
- 删除、修改操作不可持久化到宿主机
- Agent 不能通过提权或路径穿越影响宿主
- 使用成本低，不需要对每个任务启动全量容器
- 重启后沙箱状态可丢弃（临时文件自动清理）

传统 `chroot` 不具备写时隔离能力；`mount namespace` 单独使用仍可能因权限泄露而写入真实磁盘。需要一种组合式隔离方案。

## 原理：Overlay + Namespace

OpenClaw 沙箱的核心基于 Linux 内核的两个特性：**mount namespace** 与 **Overlay 文件系统**。

当启用 sandbox 后，每个 Agent session 会执行如下操作：

1. 创建一个新的 mount namespace 与 user namespace 配对，进程在该空间内拥有仿真的 root 权限，且无法影响宿主 namespace 的挂载点。
2. 在 namespace 内挂载一个 overlay 文件系统，其构成如下：
   - **lowerdir**：宿主上需暴露给 Agent 的目录，以只读方式挂载（例如 `/home/user/project`）
   - **upperdir**：一个 tmpfs 或临时目录，所有写操作、修改、新建文件都写在这里
   - **merged**：Agent 看到的最终视图，由 lower 和 upper 组合而成
3. Agent 的所有文件访问都经由 merged 目录。当 Agent 执行 `rm -rf /home/user/project` 时，内核看到的是 overlay 视图，删除操作仅会在 upperdir 中创建“whiteout”标记，原 lower 层文件完全不受影响。
4. Session 结束后，卸载 overlay 并销毁 upperdir (tmpfs)，所有脏数据消失。

配合 user namespace 的 UID/GID 映射，Agent 在沙箱内即使以 uid 0 运行，在宿主看来也只是非特权用户，无法执行真正的 `mount` 或 `pivot_root` 等危险操作。

## 配置与验证（操作步骤）

在 OpenClaw 配置文件（`config.yaml`）中启用沙箱并指定暴露目录：

```yaml
sandbox:
  enabled: true
  # 暴露给 Agent 的宿主目录（只读底层）
  paths:
    - /home/user/workspace
  # upper层，默认 tmpfs，可限制大小
  upper_fs: tmpfs
  tmpfs_size: 512m
  # 允许 session 间持久化的白名单路径（会以 bind mount 可写方式挂入）
  persist:
    - /home/user/agent-output
```

启动 Agent 后，在内执行破坏性测试：

```bash
# 进入 OpenClaw 提供的调试控制台
openclaw sandbox exec -- ls /
# 创建文件并删除
touch /home/user/workspace/secret.txt
rm -rf /home/user/workspace/*
# 退出，检查宿主目录
ls /home/user/workspace   # secret.txt 不存在，原有其他文件完好
```

重启 session 或开启新 session，上次的修改全部丢失，只有 persist 目录保留。

## 踩坑记录

1. **内核参数限制**  
   部分云虚拟机或容器内 `kernel.unprivileged_userns_clone=0`，导致无法创建新的 user namespace。需要 `sysctl -w kernel.unprivileged_userns_clone=1`，或确认 seccomp profile 没有拦截 `clone` 系统调用。在不支持的环境中，沙箱会降级为只读 bind mount 模式，功能受限。

2. **tmpfs 内存溢出**  
   如果 Agent 频繁写大文件（例如解压数据集），tmpfs 会占满内存。务必通过 `tmpfs_size` 限制大小，并监控 `/tmp/overlay-upper` 占用量。可改为基于磁盘的临时目录（如 ext4 挂载的独立分区），但会增加清理步骤。

3. **UID/GID 映射混乱**  
   在 user namespace 内，宿主文件的属主会被映射成 `nobody:nogroup`，导致 Agent 反馈“Permission denied”。解决方法：为暴露目录设置 `other read` 权限，或使用 `subuid/subgid` 映射（需配置 `/etc/subuid`）。更简单的方法是让 Agent 在沙箱内以同一虚拟 UID 运行，并预先设置宿主目录权限为容器 UID 可读。

4. **macOS 兼容性**  
   Overlay+namespace 是 Linux 特有机制。在 macOS 上 OpenClaw 使用 SIP 保护下的临时目录副本+硬链接模拟弱隔离，安全性弱于原生。生产环境建议在 Linux 服务器运行，或在 macOS 上通过虚拟机（Lima/UTM）中启用完整沙箱。

## 可复用建议

- **分层防御**：即使有沙箱，仍应限制 Agent 能看到的宿主路径，只暴露最小必要目录；配合 AppArmor/SELinux 阻止进程访问 `/proc`、`/sys` 中的敏感接口。
- **审计日志**：开启 OpenClaw 的 sandbox 审计（`sandbox.audit_log: true`），所有越过边界的操作（如尝试访问未暴露路径）都会被记录。
- **持久化策略**：避免将整个 `$HOME` 暴露为持久化目录；为 Agent 输出单独创建一个归档目录，并定时清理。
- **CI/CD 集成**：在自动化流水线中，创建独立 user namespace 并挂载 workspace 的 overlay，确保构建产物干净且无主机残留。
- **测试 check 清单**：
  1. 在沙箱中执行 `rm -rf / --no-preserve-root`，确认宿主机 `/` 未受影响
  2. 尝试 `mount -o remount,rw /`，系统调用应被禁止
  3. 检查 session 结束后 tmpfs 是否自动释放
  4. 验证持久化目录写入正常，且非持久化写入在重启后消失

## 总结

OpenClaw 的沙箱不是魔法，而是组合了 Linux 内核久经考验的隔离原语。Overlay 文件系统提供写时复制视图，mount+user namespace 锁定边界，使得 Agent 即便执行毁灭性命令，也只能在临时层里折腾。对于自动化实践者而言，这套机制在“让 Agent 自由操作”和“保证宿主绝对安全”之间找到了一个工程上可落地的平衡点。只要配置得当、踩坑有预案，你完全可以信任 Agent 在沙箱内的任何文件操作。

---

