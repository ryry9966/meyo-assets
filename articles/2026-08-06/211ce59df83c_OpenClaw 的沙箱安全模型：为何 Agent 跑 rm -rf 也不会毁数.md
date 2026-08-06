---
title: OpenClaw 的沙箱安全模型：为何 Agent 跑 rm -rf 也不会毁数据
feedId: 31873
source: 综合讨论
publishedAt: 2026-08-06
---

## 背景

Agent 工具链最让人不安的地方，不是模型“不听话”，而是它被允许执行文件操作。一旦给了 shell 执行权限或文件写入能力，一条不合适的 `rm -rf /` 就可能让工程目录消失。传统做法是信任代码、限制 prompt、事后恢复，但工程上真正可靠的是**运行时的强制隔离**。OpenClaw 的 sandbox 在设计上围绕“Agent 只能破坏它自己看到的那个世界，而那个世界是临时的、受限的、可丢弃的”这一点展开。

## 问题：误删的根源在哪

Agent 通常是工具组合：shell 命令、Python 片段、文件读写 MCP 服务。风险在于：

1. **权限过大**：Agent 进程拥有用户级权限，能修改宿主任意被他可访问的文件。
2. **路径逃逸**：通过 `../`、符号链接、绝对路径，轻易跳出预期工作目录。
3. **副作用持久化**：对真实文件系统的修改是永久的，且难以回滚。

只靠 prompt 约束是不可靠的；需要的是内核级边界，让“误删”这个动作本身就被禁止或变成无害操作。

## OpenClaw sandbox 的安全模型

OpenClaw 的沙箱不是简单 chroot，而是一套**多层防御**，核心思路是：限制视图 + 限制动作 + 限制持久化。

### 1. 文件系统视图隔离

每个 Agent 会话会创建一个私有挂载命名空间（mount namespace），内部 rootfs 通过 OverlayFS 叠加生成。Agent 看到的 `/` 根目录实际由三层组成：

- **下层（只读基础层）**：来自宿主的最小化 rootfs，或一个预先构建的镜像层，Agent 不可修改。  
- **中层（只读项目层）**：宿主指定映射的“安全目录”，如 `~/projects/current:ro`。Agent 可读，但不能写。  
- **上层（可写临时层）**：会话独有的 tmpfs 或 overlay upperdir，Agent 的所有写入、删除、重命名直接发生在这里。

这带来两个关键效果：

- Agent 运行 `rm -rf /usr /etc /home`，它删的只是 upper 层中的空目录或影子（whiteout），真实文件毫发无伤。  
- 会话结束后，上层直接被丢弃，环境回到初始状态。

### 2. 写路径白名单 + 绝对路径拒绝

文件写入不会默认允许在任意位置发生。OpenClaw 在工具调用层拦截文件操作请求（无论是 shell 动作还是 MCP 的 `write_file`），并执行路径规范化校验：

- 所有路径先调用 `cleanpath()`，消去 `..` 和多余的 `/`，再解析符号链接，得到规范化绝对路径。  
- 只允许该路径落在 `/workspace`（即 upper 层的挂载点）之下，否则直接拒绝。  
- 禁止跨设备访问：`rename()`, `link()` 等操作若目标不在同一文件系统范围内，会被 seccomp 或 LSM 拦截。

这样，即使 prompt 诱导 Agent 执行 `echo 'bad' > /etc/cron.d/evil`，请求会被网关直接返回 Permission Denied，连系统调用都不会触发。

### 3. 系统调用限制：让 rm 本身失效

单纯的文件路径拦截仍可能被绕过（例如通过直接 open() 带 O_CREAT）。OpenClaw 使用 seccomp-bpf 加载 deny list，主要禁掉：

- `unlink` / `unlinkat`：对任何非 `/workspace` 下的文件直接返回 EACCES。
- `mount` / `umount2`：禁止改变挂载结构。
- `chroot` / `pivot_root`：阻止二次逃逸。
- `chmod` / `chown` 中涉及 SUID/SGID 的操作。

对于删除操作，seccomp 过滤器检查第一个参数指向的路径前缀，非白名单内则拒绝。这正是“Agent 运行了 `rm -rf /`，结果只是 upper 层被清空”的根本保障。因为 `/` 下的真实 inode 都不在白名单路径下，删除动作被 seccomp 提前截断，唯一可触及的是 `/workspace` 下的 overlay。

### 4. 能力裁剪与用户命名空间

Agent 进程以非 root 用户运行，且位于独立的 user namespace 内，其 uid 在宿主侧映射为高位无权限的 anonymous uid。关键 Linux capabilities 被彻底移除：`CAP_DAC_OVERRIDE`、`CAP_FOWNER`、`CAP_SYS_ADMIN` 等全部 drop。这杜绝了通过 `setuid` 或 capability 提权的可能性。

## 配置与实施要点

如果你在自己的 OpenClaw 部署中启用 sandbox，关键配置项如下（示例）：

```yaml
sandbox:
  enabled: true
  engine: overlay
  workspace_size_limit: 512Mi
  mounts:
    - source: /home/user/project
      target: /mnt/project
      mode: ro
  seccomp_profile: default_deny_delete
  allowed_write_prefixes:
    - /workspace
    - /tmp
  confine_user: sandboxed
```

实际运营中，**必须**单独给 `/tmp` 分配 tmpfs 并提供配额，防止 Agent 通过大量写临时文件耗尽宿主磁盘。更严格的做法是仅允许 `/workspace` 可写，并关闭 `/tmp` 写入。

## 踩坑记录

1. **符号链接决议时机错误**：工具层如果在 open 前验路径，但 open 时另一个线程替换了符号链接，会产生 TOCTOU 竞态。必须在 open() 时使用 O_NOFOLLOW 并配合 seccomp 追踪 AT_SYMLINK_NOFOLLOW 行为。  
2. **procfs 信息泄露**：`/proc/self/fd` 可能暴露宿主文件描述符。建议完全不挂载 `/proc`，或使用 `hidepid=2` 配合 pid namespace 隔离。  
3. **Overlay 性能损耗**：大量小文件写入时 upper 层 inode 消耗快，应提前设置 `nr_inodes` 限制，并在会话结束后强制回收。  
4. **MCP 服务绕过**：如果 MCP 服务器运行在沙箱外并接受文件路径参数，Agent 可能通过 MCP 绕过 sandbox 限制。安全模型要求 MCP 插件也必须运行在同一沙箱边界内，或做路径强制校验。

## 可复用的工程建议

无论你是否使用 OpenClaw，任何允许 Agent 执行文件操作的系统都可以借鉴以下原则：

- **创建销毁式的工作区**：每个任务独立 overlay/tmpfs，结束即丢，绝对不重用。  
- **默认拒绝文件写**：除非显式声明可写目录，否则所有写操作失败。  
- **用 seccomp 兜底**：不依赖用户态路径过滤，在内核入口处阻断危险系统调用。  
- **只读挂载必要资源**：代码库、数据集、配置都 `ro` 引入，Agent 只能影响临时产物。  
- **对插件做同域安全审计**：任何能传递文件路径的接口都可能成为逃逸通道。

## 总结

OpenClaw 让 Agent 可以执行包含文件删除的任务，但并不会毁掉宿主数据，核心是因为它把“危险动作”引导到了一个**一次性且与真实文件系统隔离的视图**中。通过 mount namespace、overlay、严格路径白名单、seccomp 和 capabilities 裁剪，构成了一条完整防御链。对工程实践者来说，最值得记住的不是某一个配置项，而是“不要让 Agent 有资格破坏真实数据”这一原则，以及实现这一原则所需的组合机制。

---

