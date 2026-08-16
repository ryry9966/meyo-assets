---
title: OpenClaw sandbox 安全模型拆解：为什么 Agent 不会一不留神删掉你的文件
feedId: 33485
source: 综合讨论
publishedAt: 2026-08-17
---

## 背景

在 OpenClaw 里接 Agent、MCP 工具或自动化插件时，最常见的顾虑不是“它不够聪明”，而是“它太勤快”。一个路径解析错误、一段插件生成的清理脚本、一个 MCP server 返回的脏参数，都可能变成真实文件系统里的 `rm -rf`。

OpenClaw 的防御思路不是继续给模型加“不要乱删文件”的 prompt，而是在执行层加一层 sandbox。这个 sandbox 的核心目标是：**Agent 可以失败、可以乱试，但不应触及你明确授权之外的真实文件。**

## 问题：误删通常不是模型“坏”，而是边界模糊

实际踩过的场景大致有三类：

1. **路径拼接错误**：Agent 把 `~/project` 拼成 `~/project /`，多一个空格就是另一个目标。
2. **递归范围失控**：清理临时文件时，从 `tmp/cache/` 误扩到 `tmp/`。
3. **第三方 MCP 工具越界**：某些 MCP server 直接调用宿主文件系统，绕过了 OpenClaw 自身的文件工具。

真正有效的防护，不能只靠“让 Agent 小心点”，而要在文件系统访问、命令执行和 MCP 工具调用三个层级都设置边界。

## 做法：三层边界 + 回收站机制

在 OpenClaw 中，我建议把 sandbox 配置拆成以下三层。

### 1. 挂载与写路径默认拒绝

默认只读挂载工作区，只有显式声明的路径才允许写入。示例配置：

```yaml
sandbox:
  default_write_policy: deny
  mounts:
    - src: ./workspace
      dst: /workspace
      access: read
    - src: ./scratch
      dst: /scratch
      access: read-write
  write_allowlist:
    - /scratch
    - /workspace/.openclaw_tmp
```

这样即使 Agent 想写 `/home/user/.bashrc`，sandbox 层会直接拒绝，而不是靠模型自觉。

### 2. 删除操作先进回收站

对 `rm`、`mv`、`shred`、`dd` 等高风险操作，开启 trash/versioning。删除时不是直接 unlink，而是移动到 `.openclaw_trash/`，并保留原始路径、时间戳和 inode 信息。这样即使删错，也可以在回收站里按路径回溯。

```yaml
sandbox:
  trash:
    enabled: true
    path: /scratch/.openclaw_trash
    retain_days: 7
```

### 3. 第三方 MCP 工具单独限制

MCP server 是容易绕过 sandbox 的口子。OpenClaw 如果直接让第三方 MCP server 跑在宿主环境里，它一个 `os.remove()` 就能穿透文件工具。因此要在 MCP runner 层限制工作目录和系统调用：

```yaml
mcp:
  runner:
    workdir: /scratch/mcp
    allow_host_fs: false
    syscall_filter:
      allow: [read, write, openat, close, stat, lstat]
      deny: [unlink, rmdir, renameat2]
```

这样第三方 MCP 工具即便有恶意或 bug，也只能在受限目录内操作。

## 踩坑点

### 1. 只做 prompt 约束，不做执行层拦截

早期版本里，我试过在 system prompt 里写“禁止删除工作区外文件”。结果是模型在长任务中会忘记，或者被插件注入的指令覆盖。prompt 约束只能作为辅助，不能作为安全边界。

### 2. symlink 逃逸

如果 sandbox 只检查路径字符串，`/scratch/link` 指向真实目录时，就可能绕过限制。必须解析 realpath 后再判断。配置里要开启：

```yaml
sandbox:
  resolve_symlinks: true
  deny_mount_escape: true
```

否则一个 `ln -s /home/user /scratch/link` 就能让防护失效。

### 3. 回收站不是备份

回收站会被定时清理，或者 overlay 重置后消失。不要把 sandbox 的 trash 当成备份方案。对于重要文件，仍应在宿主机侧做版本控制或快照。

### 4. 过度信任 dry-run

dry-run 模式能拦住很多高风险命令，但部分脚本工具内部不使用 shell，而是直接调用 `unlink` 系统调用。dry-run 对这些路径无效，必须配合 syscall filter 使用。

## 可复用建议

- **先用只读模式跑一周**：让 Agent 在只读工作区里执行任务，观察它会试图写哪些路径。把这些路径逐一审核后再加入 write_allowlist。
- **高风险操作封装为工具**：不要直接暴露原始 shell。把 `delete_file`、`move_file` 等操作封装成带路径白名单和确认逻辑的工具。
- **用 bwrap/firejail 做后端验证**：如果不想一上来就上 Docker，可以用 `bwrap` 或 `firejail` 作为轻量 sandbox 后端，先验证规则是否生效。
- **审计日志要落到宿主机**：sandbox 内的操作日志要实时同步到宿主机，否则沙箱被重置后排查无据。

## 总结

OpenClaw 的 sandbox 安全模型之所以能避免“Agent 误删文件”，不是因为它更聪明，而是因为它把“能不能碰”和“怎么碰”从模型决策中剥离出来，交给执行层强制实施。

默认拒绝、写路径白名单、回收站机制、MCP runner 隔离，这四个组合起来，才能让 Agent 在受限环境中大胆试错，而不会对真实文件造成不可逆破坏。安全边界做得越靠下，你的自动化任务才能跑得越放心。

---

