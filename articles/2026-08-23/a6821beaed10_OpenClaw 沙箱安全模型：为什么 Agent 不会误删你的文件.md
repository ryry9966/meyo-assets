---
title: OpenClaw 沙箱安全模型：为什么 Agent 不会误删你的文件
feedId: 34351
source: 综合讨论
publishedAt: 2026-08-23
---

## 背景

在 OpenClaw 里，Agent 经常被允许执行 shell、读写文件、调用 MCP 工具或插件。一个很实际的担忧是：如果 Agent 理解错了指令，执行了 `rm -rf /`、`find / -name "*.tmp" -delete`，或者把 `$HOME` 当成了临时目录，会不会把宿主机文件删掉？

单靠提示词约束并不靠谱。Agent 不是每次都能稳定遵守“不要删系统文件”的要求，真正能兜底的是 sandbox 层。

## 问题

Agent 误删文件通常不是恶意，而是两类工程问题：

1. **路径幻觉**：把 `/data` 当成 workspace，把 `/home/user` 当成临时目录，或者把 `*` 扩展到不该碰的路径。
2. **工具越权**：MCP 工具或插件直接暴露了宿主机 API，绕过了文件系统限制。

所以我们需要回答的核心问题是：OpenClaw 的沙箱模型如何从机制上避免“Agent 删除宿主机文件”。

## 做法 / 步骤

OpenClaw 的沙箱安全模型可以拆成四层，工程上建议按这个顺序配置和验证。

### 1. 文件系统隔离

沙箱默认使用独立的根文件系统或 overlayfs。宿主机路径不会自动出现在沙箱里，只有显式声明的挂载才可见。删除操作通常只作用于沙箱的可写层，不会直接触碰宿主机。

例如，宿主机项目目录可以只读挂载，可写目录只保留 `/workspace` 和 `/tmp`：

```yaml
sandbox:
  engine: overlayfs
  rootfs: read-only
  writable_paths:
    - /workspace
    - /tmp
  host_mounts:
    - src: /home/user/project
      dst: /mnt/host/project
      mode: ro
```

关键点是 `mode: ro`。只要宿主机目录不是 `rw` 挂载，Agent 在沙箱里执行 `rm -rf /mnt/host/project` 只会得到权限错误，或者只删除沙箱内副本。

### 2. 权限降级

沙箱进程默认以非 root 用户运行，uid 不是 0。即使 Agent 拿到了 shell，也只能写自己有权限的路径。进一步的 seccomp/AppArmor 策略会限制 mount、chroot、pivot_root、bpf 等高权限 syscall。

这意味着即使命令很危险，执行权限也会先被 uid 和 syscall 层挡一道。

### 3. 高危命令拦截

对高风险模式做策略拦截或审批。常用规则包括：

```yaml
command_policy:
  - pattern: "rm -rf /*"
    action: ask
  - pattern: "find / -delete"
    action: deny
  - pattern: "mv /* /dev/null"
    action: deny
  - pattern: "dd of=/dev/sd*"
    action: deny
```

交互模式下可以用 `ask`，自动任务里建议直接 `deny`。这样即使前面的文件系统层没拦住，命令层也能补一刀。

### 4. 快照与回滚

对 workspace 可写层定期做快照。批量任务前手动触发一次快照，误删后可以快速恢复，而不是只能重新构建。

```yaml
snapshot_every: 10m
```

## 踩坑点

以下几个坑最容易让安全模型失效：

- **把 `$HOME` 或 `/Users` 以 `rw` 挂载**：等于主动放弃隔离。Agent 执行 `rm -rf ~/.bashrc` 会直接作用在宿主机。
- **调试时关闭沙箱忘记恢复**：`--no-sandbox`、`--privileged` 会让所有策略失效。调试完必须恢复。
- **MCP 工具绕过文件系统**：如果 MCP server 提供 `run_shell(host)` 或 `file.write(host_path)`，文件系统沙箱管不到，需要在 MCP 网关层限制工具白名单。
- **插件权限过大**：OpenClaw 插件如果声明了宿主机文件读写权限，也可能绕过沙箱。安装前要审查 manifest。
- **overlay 层膨胀**：频繁删除写入会产生大量 whiteout，占用 inode 和磁盘。需要定期清理沙箱可写层。
- **把 Docker socket 挂进沙箱**：`/var/run/docker.sock` 一旦挂入，等于给了沙箱宿主机 root 能力，非常危险。

## 可复用建议

实际使用中，可以把这些策略固化成默认模板：

1. **默认拒绝，最小授权**：只开放 `/workspace` 和 `/tmp` 可写，宿主机路径全部 `ro` 或不挂载。
2. **删除命令必须审批**：自动任务 `deny`，交互模式 `ask`，区分 profile。
3. **沙箱使用独立 uid**，不要以 root 运行。
4. **每次修改挂载配置后做越权测试**：故意让 Agent 执行 `rm -rf /mnt/host/project`，预期应被拒绝或只读。
5. **单独评审 MCP 工具和插件权限**，不要因为沙箱安全就放松工具侧限制。
6. **批量任务前手动触发快照**，保留最近 2-3 个可回滚点。

## 总结

OpenClaw 的沙箱模型能阻止 Agent 误删文件，核心不是因为 Agent 变聪明了，而是因为文件系统边界、权限降级、高危命令拦截和快照恢复组成了多层防线。

如果配置不当，尤其是把宿主机目录可写挂载、挂入 Docker socket、放行高危 MCP 工具，仍然会出事。工程上的安全是分层验证出来的，不是默认保证。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/811c2d528db29f80.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/664949fc1c57d421.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/dd2190dc8054fcbc.png)

