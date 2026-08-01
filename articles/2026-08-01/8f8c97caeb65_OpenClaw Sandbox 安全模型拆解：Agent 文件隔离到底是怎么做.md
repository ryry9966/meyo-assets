---
title: OpenClaw Sandbox 安全模型拆解：Agent 文件隔离到底是怎么做到的
feedId: 31222
source: 综合讨论
publishedAt: 2026-08-01
---

# OpenClaw Sandbox 安全模型拆解：Agent 文件隔离到底是怎么做到的

## 背景：Agent 离你的文件有多近？

MCP、Skill、Code Interpreter、本地自动化……OpenClaw 社区里越来越多的 Agent 被赋予了读写本地文件的能力。一个 “帮我整理桌面” 的指令，理论上可以变成 `rm -rf ~/Desktop/*`，Agent 的一句误解，可能就是一次数据灾难。

OpenClaw 在设计之初就面临这个问题：**怎么让 Agent 有足够的能力完成任务，又不让它真正碰你的原始文件？** 答案是一套从文件系统层面构建的沙箱安全模型。这篇文章会把它的设计和实现细节拆开来看，适合正在做 Agent 安全封装或自建插件的同学参考。

## 问题：传统运行方式的风险

如果只是拿系统命令执行器跑一句 `ls`，用户能做的防护非常有限：

- `subprocess.run(["rm", "-rf", path])` —— 一旦 prompt 诱导失败，文件就真没了。
- Agent 可能在 .bash_history、配置目录里拉屎，污染环境。
- 即使加了白名单检查，在代码层面依然存在绕过风险（符号链接、路径穿越）。

所以 OpenClaw 的思路不是 “在调用之前检查”，而是 **“从调用那一刻起，Agent 看到的就不是你的真实文件系统”**。

## OpenClaw 的做法：多层隔离沙箱

OpenClaw 的 sandbox 并非简单的 Docker 包装，它采用了一套内建的 **overlay 文件系统 + 命名空间隔离 + 系统调用过滤** 的组合方案，核心逻辑固化在 runner 层，而不是靠用户手动挂载。

### 1. Overlay 文件系统：只读底 + 临时上

每个 Agent 会话启动时，OpenClaw 会为它创建一个独立的文件系统视图：

- **Lower layer**：用户指定的工作目录（如 `/home/user/project`）以只读方式绑定进一个 overlay 挂载点。
- **Upper layer**：一个 `tmpfs` 内存临时层，所有写操作都在这里发生。
- **Merge view**：Agent 进程实际看到的是两者的联合挂载，可以 “修改” 文件，但修改仅在临时层生效，不会回写到原目录。

当你允许 Agent 执行文件操作时，它删除或改动的只是 tmpfs 里的副本，真实文件原封不动。

### 2. 白名单路径的 bind mount

总有些场景需要 Agent 产出持久化结果，比如生成报告、下载文件。OpenClaw 提供了一个 `allowed_mounts` 配置，允许用户声明若干个 “可写出口”：

```
sandbox:
  allowed_mounts:
    - host_path: /home/user/reports
      box_path: /workspace/output
      writable: true
```

实现上，OpenClaw 在创建新的 mount namespace 后，会在 overlay 联合挂载之上，再以 bind mount 的方式把白名单路径挂进去。这样 Agent 认为自己直接写 `/workspace/output`，实际上写到了宿主机受控的目录。

### 3. Seccomp 系统调用过滤

文件系统隔离能防 “假删除”，但无法阻止 Agent 尝试执行 `unlink` / `rmdir` / `rename` 等在真实文件系统上的破坏性调用。OpenClaw 的 sandbox 预置了一个 seccomp BPF 过滤器：

- 默认拦截所有可能导致真实文件变动的系统调用。
- 通过 agent 进程的 pid 和 mount namespace 上下文，允许对临时层或白名单路径的合法操作，其余一律返回 EPERM。

这意味着即使有人在 prompt 里精心构造一个绝对路径 `../../etc/passwd`，Agent 也无法删除它——因为真实的 `/etc` 根本没挂进这个 sandbox 视图，且 seccomp 直接拒绝了对该路径的 unlink。

## 踩坑实录：三个真实问题

### 坑1：pivot_root 失败

早期版本试图用 `pivot_root` 直接替换 Agent 的根文件系统，但在某些内核配置下会报 `EINVAL`。根本原因是 `pivot_root` 要求 **新根和旧根不在同一个挂载树上**，而初始代码把临时根放在了同一个 `/tmp` 里。解决办法是创建一个新的 mount namespace（`CLONE_NEWNS`），并在其中把 `/` 重新挂载为 private，然后再执行 pivot_root。

**可复用建议**：除非你在做容器 runtime 开发，否则用 `chroot` 或直接利用 overlay 联合挂载 + `unshare` 更容易维护。

### 坑2：符号链接逃逸

Agent 在 sandbox 里创建符号链接指向 `/mnt/real_secret`，如果 overlay 的 mount 参数不当，这个链接解析时会突破到宿主机。OpenClaw 的解决方式是：

- 在 mount 时加 `nosymfollow` 选项（需要内核支持）。
- 同时限制 `procfs` 的 `/proc/self/fd` 访问，防止 fd 方式逃逸。
- 对允许写出的白名单路径，也强制禁用符号链接创建。

### 坑3：白名单路径父目录缺失

当 `allowed_mounts` 中声明的 `box_path` 深层目录在 upper layer 中不存在时，bind mount 会直接失败。必须在 mount 前确保路径存在，OpenClaw 会在 tmpfs 上预先 `mkdir -p` 创建这些目录。

## 可复用建议：如果你需要自己实现

1. **优先使用 rootless 方案**：社区有很多替代，比如 Podman、Firescope，或者直接用 `bwrap`（bubblewrap）作为轻量级 sandbox。借助成熟工具比重复造轮子更安全。
2. **默认不可写是铁律**：即使需要持久化，也要让持久化路径显式声明，不接受 Agent 的任何动态路径请求。
3. **不要只依赖路径检查**：沙箱必须基于命名空间+系统调用过滤，纯 Python if-else 判断一个路径是否是白名单路径，是不可靠的。
4. **写一份 seccomp 模板**：可以在 OpenClaw 社区找到官方模板，覆盖 `unlink`, `unlinkat`, `rmdir`, `rename`, `creat` 等调用，配置一次，全局生效。

## 总结

OpenClaw 的 sandbox 安全模型可以浓缩为三层：

- **文件视图隔离**：overlay 只读底 + tmpfs 上，让 Agent 改不动真文件。
- **白名单出口**：bind mount 把持久化需求限制在用户指定的受控路径。
- **系统调用兜底**：seccomp 拒绝一切不在白名单上的破坏性操作。

这些技术不算新，但被工程化地整合到 Agent 运行时里，就成了 “为什么 Agent 不会误删文件” 的真正答案。对于社区里正在做自动化工具的各位，这套模型完全可以拆出来，用到你们自己的插件或 MCP 服务里去。

---

