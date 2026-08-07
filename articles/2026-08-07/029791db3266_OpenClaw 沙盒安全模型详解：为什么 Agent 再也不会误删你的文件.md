---
title: OpenClaw 沙盒安全模型详解：为什么 Agent 再也不会误删你的文件
feedId: 31976
source: 综合讨论
publishedAt: 2026-08-07
---

## 背景：Agent 的文件操作噩梦

在 MCP（Model Context Protocol）驱动的 Agent 实践里，一旦给了工具执行能力，最让运维心跳加速的场景往往是：Agent 接到一个模糊指令，比如“清理临时文件”，然后一把 `rm -rf /tmp/*` 下去，不小心把宿主机的某些挂载点也清空了。或者更糟糕的，文件操作工具链没有做好参数校验，直接删除了项目根配置。

传统的缓解手段无非两种：
- **Prompt 约束**：反复在 system prompt 里写“不要删除重要文件”，但大模型依然会产生不可预知的推理路径。
- **权限最小化**：单独创建低权限用户，配合 Linux capabilities 或文件系统权限（如 `chmod 555`）限制写入和删除。这种方法面临维护困难和粒度太粗的问题：Agent 确实需要写入和删除某些临时文件，只读挂载又不够灵活。

这引出了一个核心问题：**怎样让 Agent 以为自己在操作系统文件，实际上却不会对原始文件造成任何破坏？**

## 问题分析：为什么“操作”不等于“破坏”

从工程角度看，Agent 执行的文件操作本质上是一系列系统调用：`open`、`write`、`unlink`、`mkdir` 等。如果我们能够在不修改 Agent 执行逻辑的前提下，将这些系统调用产生的副作用隔离到一个临时层里，让 Agent 仍然看到完整的文件树，但在它“删除”某个文件时仅将删除操作记录在该临时层，而原始文件毫发无损，那就能在机制上杜绝误删。

这就是 OpenClaw 的 sandbox 安全模型的起点——基于文件系统联合挂载（Union Filesystem）的写时复制隔离。

## 做法：OpenClaw 沙盒的工作机制

OpenClaw 并不是一个简单的文件代理，它利用 Linux 内核的 OverlayFS 技术，为每个 Agent 的运行时构造一个私有的文件系统栈。该栈由三层构成：

1. **只读层（lowerdir）**  
   指向主机上希望 Agent 访问的目录，例如 `/home/user/project` 或整个 `/data` 目录。该层挂载为只读，拒绝任何直接写入。

2. **工作层（workdir）**  
   一个高速存储位置，用于 OverlayFS 内部准备文件时使用，对 Agent 不可见。

3. **可写层（upperdir）**  
   所有 Agent 对文件系统的修改、创建、删除都被实际写入这一层。`unlink` 操作会在可写层生成一个“白洞”（whiteout）文件，表示该文件已被删除。原始文件在只读层原封不动。

最终，Agent 看到的是一个合并后的视图（merged 目录）。当 Agent 执行 `rm /project/important.yaml` 时，`important.yaml` 从合并视图中消失，但只读层中的原始文件仍保留，只是被可写层的 whiteout 掩盖了。

**配置示例（`openclaw.yaml`）**：
```yaml
sandbox:
  enabled: true
  type: overlay
  lower: /data/projects      # 只读保护目录
  upper: /var/lib/openclaw/sandbox/agent-01/upper
  work: /var/lib/openclaw/sandbox/agent-01/work
  merged: /sandbox/agent-view    # Agent 看到的根目录
  readonly_paths: ["/etc", "/usr"]
  writable_tmp: true
```

启动 Agent 后，其所有工具（如命令执行、文件读写插件）都会被 `chroot` 到 `/sandbox/agent-view` 或者通过 mount namespace 隔离，确保它只能接触这个合并视图。恶意或错误的 `rm -rf` 只会污染 upper 层，停止 Agent 并丢弃 upper 层后，原始数据即刻恢复。

### 与 MCP 工具的结合

在 MCP 服务器实现文件系统工具（如 `fs.read`、`fs.write`、`fs.delete`）时，通常会直接调用主机文件路径。OpenClaw 通过将工具的根路径映射到 merged 目录，无需修改工具代码即可实现隔离。你只需要在 MCP 服务器配置中将工作路径设置为 `/sandbox/agent-view` 即可，Agent 甚至感知不到自己被关在沙盒里。

## 踩坑点：沙盒不是银弹

虽然 OverlayFS 解决了核心的删除问题，实践中还是有几个容易踩的坑。

**1. 硬链接与 inode 泄露**  
如果 Agent 通过 `link` 系统调用为只读层的文件创建了硬链接，这些硬链接也将反映在可写层。一旦 Agent 通过硬链接访问并修改文件内容，upper 层会生成完整的文件副本，出现“写时复制”现象。这本身不会破坏原始文件，但会消耗 upper 层磁盘空间，如果不监控，可能导致磁盘被写满。

**2. `unlinkat` 与目录删除的 whiteout 不彻底**  
某些文件操作（尤其是使用 `AT_REMOVEDIR` 标志的 `unlinkat`）在旧版本内核中与 OverlayFS 的 whiteout 机制配合不佳，可能造成“目录看起来空了但无法完全删除”的怪异状态。解决方案是确保宿主机内核版本不低于 5.4，并使用 OpenClaw 推荐的 mount 参数，如 `redirect_dir=on`。

**3. 嵌套容器环境冲突**  
如果在 Docker 容器内再使用 OpenClaw 的 OverlayFS 沙盒，会遇到 OverlayFS 不支持嵌套挂载的问题（`overlay` 模块默认不允许在 overlay 之上再挂载 overlay）。此时需改用 fuse-overlayfs 作为后端，或使用 `--privileged` 给予容器额外挂载能力（不推荐生产环境）。OpenClaw 支持配置使用 fuse-overlayfs 作为平替。

**4. 符号链接逃逸**  
Agent 可能通过创建指向合并视图之外的符号链接来尝试“逃逸”。OpenClaw 默认启用了符号链接重定向检查，所有创建 symlink 的操作都会验证目标是否在 merged 目录之内，否则拒绝创建。但如果你的 MCP 工具链允许 Agent 直接执行系统命令，务必同时开启 no_new_privs 和禁止 mount 的操作。

## 可复用建议

1. **分 Agent 使用独立 upper 层**  
   不要让多个 Agent 共用同一个 upper 目录，否则互相干扰，且丢弃操作不方便。可以为每个任务或会话创建唯一的 upper，任务结束后整体删除。

2. **设置磁盘配额**  
   对 `/var/lib/openclaw/sandbox` 所在文件系统设置 XFS 配额或使用 systemd 的 `TemporaryFileSystem` 限制大小，避免写时复制撑满磁盘。

3. **监控 whiteout 文件积累**  
   定期检查 upper 层的 whiteout 数量，它可以反映 Agent 误删除的频次，用于改进 prompt 或工具定义。你可以通过 `find upper/ -type c -o -type l | wc -l` 简单统计。

4. **关键工作流双重保险**  
   对于极度重要的文件（如数据库备份），即使有沙盒隔离，仍然建议在 Agent 工具定义里加上人工确认环节（如 `require_confirmation: true`），防止沙盒 whiteout 掩盖真实问题导致后续误判。

5. **只读挂载敏感系统目录**  
   配置中将 `/etc`、`/boot`、`/sys` 明确定义为 `readonly_paths`，即使 merged 视图允许写入，OpenClaw 仍会以 `MS_RDONLY` 重新挂载这些路径，杜绝任何意外。

## 总结

OpenClaw 的 sandbox 安全模型从文件系统层面对 Agent 的操作施加了强隔离，将“删除”动作转化为可逆的影藏操作，加上 chroot / namespace 的约束，误删文件的风险被降低到了工程上可控的水平。它不依赖大模型的“理解”，而是一场机制上的防御。

当然，沙盒无法解决所有安全问题，比如 Agent 读取敏感文件后通过 API 外传数据，还需要配合 eBPF 网络监控和对工具返回值的审查。但就防止误删文件而言，基于 OverlayFS 的写时复制隔离是当前成本最低、可靠性最高的方案之一。如果你正在搭一个 Agent 需要碰真实数据的实验环境，建议从 OpenClaw 的沙盒配置开始，而不是事后 `debugfs` 救火。

---

