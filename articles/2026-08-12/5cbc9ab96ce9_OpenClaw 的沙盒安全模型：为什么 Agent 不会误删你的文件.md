---
title: OpenClaw 的沙盒安全模型：为什么 Agent 不会误删你的文件
feedId: 32762
source: 综合讨论
publishedAt: 2026-08-12
---

## 背景

在任何让 Agent 直接操作本地文件系统的场景里，“误删文件” 都是第一个被翻出来的安全问题。用户担心的点很实在：如果 Agent 执行了一段我根本没审查过的 shell 命令，或者某个 MCP 工具返回了不可信的路径，它会不会顺手把 ~/.ssh 或者整个项目目录给删了？

OpenClaw 从一开始就在架构层面对这种风险做了控制，并不是简单地在提示词里加一句“请不要删除用户文件”，而是用了一个可配置、可审计的 Sandbox 层。这个 Sandbox 主要借助 Linux 的 mount namespace 与 seccomp 机制，把 Agent 的执行环境关进一个受限的文件系统视图里，再通过 MCP 服务器与宿主之间做白名单式交互。

## 问题：Agent 操作文件的真实风险

大多数 Agent 框架允许通过 `execute_command` 或内置工具读写文件。在没有沙盒的情况下，Agent 一旦被 prompt injection 或者控制指令误导，就可能执行类似以下操作：

- `rm -rf /home/user/projects/*`
- 覆盖 `~/.bashrc`
- 通过 MCP 文件工具遍历并删除你能访问的一切路径

哪怕是不经意的错误，比如在清理缓存时用了错误的通配符，后果也很严重。因此我们需要一种机制，让 Agent 即便“想做坏事”也做不到，而不是指望模型每次都做出安全决策。

## 核心机制：文件系统视图隔离 + 能力白名单

OpenClaw 的 Sandbox 不依赖 Docker 重量级容器，而是在进程级别构建隔离环境。其核心由三个部分组成：

1. **Mount namespace + 只读根文件系统**  
   Agent 进程启动时会 clone 一个新的 mount namespace，基础文件系统被挂载为只读。宿主的实际工作目录通过 `bind mount` 注入到一个固定的沙盒路径（例如 `/sandbox/workspace`），并且可以设定为读写或只读。Agent 只能看到这个受限的目录树，宿主其他敏感路径天生不可见。

2. **Seccomp 过滤危险系统调用**  
   文件删除本质上依赖 `unlink`/`unlinkat` 等系统调用。OpenClaw 在 seccomp-bpf 规则中对 Agent 进程禁用了这类系统调用。即便 Agent 记住了某个宿主路径，由于其文件系统视图里根本没有那个路径，再加上对应 syscall 被内核拦截，直接删除本地文件的可能性降为零。

3. **MCP 工具的能力白名单**  
   OpenClaw 内置的文件操作 MCP 服务器（如 `filesystem`, `workspace`）默认仅工作在 `/sandbox/workspace` 下，并且代码层面对所有路径参数做了 `canonicalize` 及前缀检查。任何试图通过 `../../../etc/passwd` 跳出沙盒的路径都会被拒绝。对文件删除操作，MCP 会额外要求用户授权或直接禁用 `delete_file` 能力，除非显式开启。

## 配置实践：从零搭建一个沙盒化 Agent

实际使用中，你只需要在 `agent.yaml` 中定义 workspace 与权限策略，即可获得默认的安全保护。

```yaml
sandbox:
  type: "mountns"
  workspace:
    host_path: "/home/user/my-agent-workspace"
    mount_path: "/workspace"
    mode: "readwrite"
  allowed_syscalls:
    - read
    - write
    - openat
    - close
    - stat
  mcp:
    - name: filesystem
      allowed_actions: ["read", "write", "list"]
      # delete 不在列表中，因此文件删除被直接禁用
```

当你通过 OpenClaw 启动 Agent 后，Agent 只能读写 `/workspace` 下的内容，运行的命令也无法越过这个边界。如果你需要只读访问某个大型数据集目录，可以再加一个 `readonly` 的 bind mount：

```yaml
  extra_mounts:
    - host_path: "/data/datasets"
      mount_path: "/datasets"
      mode: "ro"
```

## 踩坑点

**1. 符号链接穿越**  
如果宿主的 workspace 目录里存在指向外部的符号链接（例如 `ln -s /etc configs`），Agent 有可能通过跟随链接来触碰外部文件。应对方法：在挂载 workspace 之前检查并移除符号链接，或使用 `MS_NOSYMFOLLOW` 挂载选项（需要内核支持）。OpenClaw 会在沙盒初始化时自动检测软链接风险并报警。

**2. `/tmp` 和共享内存的泄漏**  
Agent 进程仍然共享宿主的部分 `tmpfs`。如果不做额外处理，Agent 可能向 `/tmp` 写入大量数据导致宿主机磁盘或内存耗尽。建议在 mount namespace 中挂载一个独立的 tmpfs 到 `/tmp`，大小限制为 64MB 之类。

**3. 临时放宽权限后的残留**  
调试阶段可能有人图省事把 `allowed_actions` 设成 `["*"]`，然后忘记改回去。这会让安全策略形同虚设。使用版本管理并把沙盒配置纳入 CI 检查，可以避免上线时还是全开权限。

**4. 部分插件要求 `unlink`**  
有一些社区插件（如自动清理旧日志）确实需要删除文件的能力。这时可以启用 `conditional_delete`：仅允许在 `/workspace/.trash` 路径下执行删除操作，相当于为 Agent 提供一个安全的“回收站”策略。

## 可复用建议

- **最小挂载原则**：只注入 Agent 真正需要的目录，读不需要的就别挂，能只读就别给写。
- **删除操作默认关闭**：除非业务有强需求，否则永远不在 MCP 工具里开放删除权限。通过覆写 `delete_file` 的 handler 返回错误。
- **日志审计**：开启 OpenClaw 的 syscall violation 日志，定期检查是否有 Agent 尝试执行被禁止的系统调用，这有助于发现 prompt injection 行为。
- **分环境配置**：开发环境可以开一个“软警告模式”，只记录违规而不杀进程；生产环境必须严格 deny。

## 总结

OpenClaw 的沙盒安全模型并不是一种“魔法”，而是通过 Linux 内核特性与 MCP 权限控制组合实现的纵深防御。它从根本上切断了 Agent 误删文件的技术路径：Agent 看不见宿主文件，删文件的系统调用又被内核禁止，即便通过 MCP 发出的文件操作也受到路径检查和能力白名单的双重限制。对于大多数个人使用和团队试点项目，这套开箱即用的 sandbox 已经足够可靠。再结合合理的工作流设计（例如重要数据备份、代码版本控制），你完全不用担心 Agent 会悄悄抹掉你的关键文件。

后续如果有兴趣，可以进一步阅读 OpenClaw 源码中 `sandbox/mountns` 和 `seccomp/profile.go` 的实现，所有安全策略都是可扩展的，你甚至可以为特定 Agent 定制自己的 seccomp 规则。

---

