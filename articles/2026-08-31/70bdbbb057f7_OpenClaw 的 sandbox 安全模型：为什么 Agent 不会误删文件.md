---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 35440
source: 综合讨论
publishedAt: 2026-08-31
---

## 背景

在 OpenClaw 这类 Agent 框架里，模型经常通过工具调用执行 shell 命令、读写文件、访问 MCP 外部服务。比如让它“整理下载目录”“清理临时文件”“批量重命名”。默认如果 Agent 进程运行在与用户相同的文件系统权限下，一个错误的 prompt 或 prompt 注入就可能执行 `rm -rf ~/` 或覆盖关键配置。实际上很多 Agent 事故都源于文件系统权限过大。

OpenClaw 的应对是把所有 Agent 执行动作放进一个受控的沙箱（sandbox）里。这个 sandbox 通常基于 Linux 内核的 namespace、seccomp、filesystem bind mount 和只读挂载组合而成，目标很简单：Agent 只能碰你明确允许它碰的目录，其他路径要么不可见，要么只读，要么直接不存在。

## 问题：为什么直接跑会误删

如果没有 sandbox，Agent 的 tool executor 一般直接调用 subprocess 或 `bash -c`。此时它继承了宿主进程的全部权限：可读 `/etc/passwd`、可写 `~/.ssh`、可执行 `rm -rf /`。即使模型没有恶意，也可能因为上下文误解或指令歧义发生危险操作。例如“删掉所有 .tmp 文件”在根目录下执行就会灾难。

OpenClaw 的解决办法是把每个命令或 MCP 调用包装进一个受限执行环境。以常见配置为例，它使用 bubblewrap（bwrap）或类似工具创建新的 mount namespace，然后：

- 把宿主根文件系统以只读方式重新挂载（`--ro-bind / /`）
- 白名单几个可写目录，如 `/tmp`、`/workspace`、`/home/user/.cache`（`--bind` 或 `--tmpfs`）
- 禁用网络或限制到特定网卡（`--unshare-net`）
- 用 `--unshare-pid` 让 Agent 进程看不到宿主其他进程
- 加 seccomp 过滤，禁止 `mount`、`reboot`、`swapoff` 等危险系统调用

如果你在 OpenClaw 的 sandbox 配置里开了 `enabled: true`，但没有指定 `writable_paths`，那么整个文件系统对 Agent 来说都是只读的。此时即使模型执行 `rm -rf /`，内核会返回 `Read-only file system`，什么也删不掉。

## 做法/步骤：在 OpenClaw 中启用并验证 sandbox

下面是一个典型的复现步骤（假设你已在 Linux 环境部署 OpenClaw，并且它支持 sandbox 配置项；不同版本配置键可能略有差异，以官方文档为准）。

1. 编辑 `config.yaml` 或等效配置文件：

```yaml
sandbox:
  enabled: true
  backend: bubblewrap
  writable_paths:
    - /tmp
    - /home/user/agent_workspace
  read_only_paths:
    - /etc
    - /usr
  network: false
  seccomp: true
```

2. 创建白名单目录并给 Agent 用户权限：

```bash
mkdir -p /home/user/agent_workspace
chown user:user /home/user/agent_workspace
```

3. 重启 OpenClaw 服务。

4. 验证：在聊天里让 Agent 执行 `rm -rf /etc/hostname` 或 `touch /etc/test_file`。观察返回：

```
Permission denied / Read-only file system
```

5. 再让 Agent 在 `/home/user/agent_workspace` 下创建文件，应该成功。

6. 查看 sandbox 日志（通常位于 `/var/log/openclaw/sandbox.log` 或通过 `journalctl -u openclaw`）确认命令被包装进了 bwrap。

## 踩坑点

- **白名单路径过宽**：如果你把整个 `/home/user` 设为 writable，等于没设。最小化可写目录，每个任务最好使用独立临时目录。
- **MCP 服务器穿透**：某些 MCP 工具（如文件管理器）可能在宿主进程里直接操作文件，不经过 Agent 的 sandbox。如果 MCP 服务器本身有 root 权限，Agent 依然可以间接删除文件。所以 MCP 服务器也要限制运行用户和目录。
- **用户命名空间被禁用**：容器或部分 VPS 默认 `kernel.unprivileged_userns_clone=0`，bubblewrap 会报错。需要开启或改用其他后端（如 firejail、landlock）。在 Kubernetes 里还涉及 seccomp profile 和 AppArmor。
- **动态库缺失**：使用 bwrap 时如果只 ro-bind 了部分目录，某些命令可能因找不到 `/lib` 或 `/usr/lib` 而失败。建议把整个 `/usr`、`/lib`、`/lib64`、`/bin` 都只读挂载，而不是只挂载个别文件。
- **性能**：每次命令都创建新 namespace 有开销。高频短命令场景（如循环调用）可以复用 sandbox 实例或使用更轻量的 landlock LSM。

## 可复用建议

- **分层防护**：不要只依赖 sandbox。同时限制运行 Agent 的系统用户权限（非 root）、使用 systemd 服务文件里的 `ProtectSystem=strict`、`ReadWritePaths=` 等。
- **默认拒绝**：宁可让 Agent 报错，也不要一开始开大权限。需要写入时再加白名单。
- **任务级临时目录**：可以在每次会话开始时创建 `/tmp/agent-$RANDOM`，把工作目录指向那里，任务结束后自动清理。
- **审计**：开启命令日志，记录每次工具调用的完整命令、参数和退出码。便于事后分析。
- **测试**：写一个专门的“破坏性测试”prompt 集，包含 `rm`、`mv`、`dd`、`mkfs` 等命令，在 CI 里跑，确认沙箱拦截。

## 总结

OpenClaw 的 sandbox 安全模型并不神秘，它就是用 Linux 内核已有的隔离机制，把 Agent 的执行环境从宿主文件系统中隔离开。默认只读、白名单可写、禁止网络和危险系统调用，这样即使模型误判或 prompt 被注入，最坏结果也就是在当前工作目录里制造一些垃圾文件，而不会删除家目录或系统配置。工程实践上，关键是控制 `writable_paths` 的最小范围，并注意 MCP 服务器和宿主运行的权限。把这些配置做成模板，每个新项目直接复用，可以显著降低 Agent 自动化带来的文件破坏风险。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/e128195435ed6a14.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/9dd00f7cab8e65d5.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/9555dc0a10e62ed3.png)

