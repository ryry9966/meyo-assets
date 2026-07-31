---
title: OpenClaw Sandbox 安全模型深度解析：Agent 为何删不掉你的关键文件
feedId: 31097
source: 综合讨论
publishedAt: 2026-07-31
---

# OpenClaw Sandbox 安全模型深度解析：Agent 为何删不掉你的关键文件

## 背景：当 Agent 获得文件系统写入权

在基于 OpenClaw 构建的自动化任务中，Agent 需要通过工具或 MCP 插件读写文件。一旦赋予 `write` 或 `delete` 权限，就面临一个真实风险：Prompt 的模糊意图、模型幻觉或插件逻辑缺陷，可能导致 **误删宿主关键文件**。传统做法是“人工确认”或“只给只读权限”，但这会限制 Agent 能力。我们需要一种机制，让 Agent 能够自由操作文件，同时从根本上隔离子系统，使其无法触及宿主机敏感区域。

OpenClaw 的 Sandbox 安全模型正是为此设计。它不是简单的路径黑名单，而是通过 **命名空间隔离 + 挂载点白名单 + 能力限制** 的三层防御，将 Agent 运行时锁在一个虚拟文件系统视图里。

## 问题定义

假设一个自动化 Agent 需要清理 `/workspace/output` 下的临时文件，但 Prompt 出现歧义，Agent 误将 `rm -rf /workspace` 理解成清理目标。如果没有沙箱，整个工作目录都会被删除。或者 Agent 通过路径遍历漏洞访问 `/etc/passwd`，后果更严重。

核心矛盾：Agent 需要局部写入能力，但又不能拥有宿主文件系统的完整视角和操作权限。

## 做法：三层防御的沙箱模型

### 1. 文件系统命名空间隔离

OpenClaw 要求 Agent 运行在独立的 Linux namespace 中（使用 `unshare` 或直接集成到 executor）。通过 `CLONE_NEWNS` 挂载命名空间隔离，Agent 看到的是一个全新的挂载树，与宿主机的 `/` 完全解耦。

启动时，OpenClaw 会为每个 Agent 创建一个私有挂载点，类似于：

```
openclaw sandbox init --root /var/lib/openclaw/sandboxes/agent-01
```

该目录成为 Agent 的根文件系统，内部只包含运行时必要的最小文件集。宿主机 `/etc`、`/home` 默认不会被挂载进去，Agent 根本没有访问这些路径的“门把手”。

### 2. 白名单挂载与权限裁剪

Agent 可能需要访问特定数据目录，比如项目目录 `/data/project`。这时通过 **bind mount 白名单** 精确注入：

```
mount --bind /data/project /var/lib/openclaw/sandboxes/agent-01/workspace/project -o nodev,noexec,nosuid
```

关键参数：
- `nodev,noexec,nosuid`：进一步限制可执行文件、设备文件与 suid 位，防止提权。
- 只读挂载可附加 `ro` 选项，例如日志目录挂载为只读，避免 Agent 篡改审计日志。

注意：白名单是**精确路径绑定**，而非整棵父目录树。即使 Agent 的 cwd 在 `/workspace/project/subdir`，若 `/workspace/project` 是绑定挂载的根，它无法通过 `../` 穿越到宿主机 `/data` 下的其他目录，因为挂载点本身就是边界。

### 3. 能力集与系统调用过滤

仅靠文件系统隔离不够，一个拥有 `CAP_SYS_ADMIN` 的进程可能重新挂载文件系统以突破隔离。OpenClaw 的 executor 会 drop 所有非必要 Linux capabilities，并配合 seccomp 过滤：
- 禁止 `mount`, `umount`, `pivot_root` 系统调用。
- 仅允许与沙箱内挂载点相关的 `openat`, `unlink` 等操作，但不允许修改挂载属性。

这样就构成了完整闭环：Agent 进程认为自己在完整的文件系统中操作，但实际上只能看到并修改白名单内的文件，且无法扩展这份视图。

## 实战配置示例

以 OpenClaw v0.4 为例，配置一个用于文档处理 Agent 的沙箱，要求可读 `/shared/models`（只读），可写 `/scratch/docs`：

```yaml
# openclaw-sandbox.yaml
sandbox:
  root: /var/lib/openclaw/sandboxes/doc-agent
  default_mount_options: "nodev,noexec,nosuid"
  mounts:
    - src: /shared/models
      dst: /models
      options: "ro,bind"
    - src: /scratch/docs
      dst: /workspace
      options: "rw,bind"
      ensure_dir: true
  capabilities: []
  seccomp_profile: "default-block-mount"
```

启动 Agent 时，OpenClaw 自动创建 `/var/lib/openclaw/sandboxes/doc-agent` 作为根，并将上述两个目录绑定到沙箱内对应的挂载点。Agent 感知到的根目录结构类似：

```
/             (sandbox root)
├─ models/    (read-only, from /shared/models)
├─ workspace/ (read-write, from /scratch/docs)
└─ ...        (最小化运行时文件)
```

如果 Agent 脚本执行 `rm -rf /`，至多会删掉 `/workspace` 下的文件，无法触及宿主机的 `/shared/models`（因为它是只读挂载，删除会被拒绝），更不会影响宿主机的根文件系统。

## 踩坑点与避坑指南

### 坑1：ensure_dir 缺失导致挂载失败

若目标目录 `/scratch/docs` 不存在，且未配置 `ensure_dir: true`，bind mount 会失败，Agent 启动异常。常见于容器初次部署或临时目录被清理后。建议在配置里显式 `ensure_dir: true`，或使用 tmpfs 动态目录。

### 坑2：嵌套挂载下的路径泄漏

如果先 bind 了整个 `/data`，再 bind `/data/project` 到子路径，Agent 可能通过 `../` 访问到 `/data` 下的其他内容。**白名单设计务必扁平化**，每个挂载点指向最细粒度的叶子目录，而非父目录。

### 坑3：Agent 依赖动态库无法运行

最小化根文件系统可能导致 Agent 进程缺少必要的动态链接库（如 glibc）。可预置一份微型 rootfs（如基于 Alpine 的 rootfs），或使用静态编译的 Agent 执行文件。OpenClaw 支持通过 `rootfs_image` 字段指定基础镜像，自动完成 provision。

### 坑4：日志与监控被隔离

若将 `/var/log` 完全隔离，审计日志会丢失 Agent 的操作记录。建议单独挂载一个日志目录（如 `/sandbox-logs`），并通过宿主机的 logging agent 采集，而非直接在沙箱内写入。

## 可复用建议

1. **最小权限原则**：每个 Agent 只挂载必需的目录，并优先使用只读挂载。
2. **专用用户映射**：在沙箱内使用非 root 用户（UID 映射），配合 `user namespace` 进一步限制，即使 Agent 逃逸出文件系统隔离，其宿主机权限也受限。
3. **临时文件使用 tmpfs**：对于无需持久化的临时输出，挂载 `tmpfs`，避免残留文件占用磁盘且可自动清理。
4. **不可变挂载点**：在关键挂载选项中加入 `ro` 和 `nodev`，可有效抵抗误删和提权。
5. **生命周期管理**：Agent 执行完毕后应立即销毁沙箱，通过 `openclaw sandbox purge --older-than 1h` 定期清理，防止沙箱泄露占用 inode。

## 总结

OpenClaw 的沙箱安全模型不是简单的“检查路径字符串”，而是利用 Linux 内核的 namespace 与挂载绑定机制构建 **真实文件系统视图的牢笼**。每次 Agent 操作文件时，内核在 VFS 层直接保证它无法跳出预定义的挂载边界，任何穿越尝试都会得到 ENOENT 或 EACCES。这使得“误删文件”变得几乎不可能——因为 Agent 根本看不到那些它无权接触的文件。

对于工程师而言，理解并善用这一模型，就能在赋予 Agent 强大自动化能力的同时，守住系统的安全底线。不必再为 `rm` 命令心惊胆战。

---

