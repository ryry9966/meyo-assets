---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 33389
source: 综合讨论
publishedAt: 2026-08-16
---

# OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件

## 背景

Agent 类工具最常见的翻车场景，不是回答错误，而是误操作文件系统。让 Agent“整理当前目录”，它可能执行 `rm -rf ~/data`；接入一个 MCP 文件服务，结果把整个工作区删掉。很多框架只靠 prompt 约束，但 LLM 并不能稳定理解“删除”的边界。OpenClaw 的 sandbox 设计思路很直接：**不把安全寄托在模型自觉上，而是在执行层做强制拦截。**

这篇文章面向已经在用 OpenClaw、MCP、插件或自动化脚本的实践用户，说明 sandbox 如何阻止 Agent 误删文件，以及配置时容易踩的坑。

## 问题

为什么 Agent 会误删？因为对 Agent 来说，删除只是工具调用之一。模型看到 `delete_file` 或 `rm` 工具，只会根据上下文判断是否该调用，而不会真正理解后果。如果工具权限给得过大，误删就是概率问题，不是能力问题。

OpenClaw 的 sandbox 不是“禁止所有写入”，而是分层控制：文件系统层、命令执行层、MCP 工具层。下面按实际配置步骤说明。

## 做法/步骤

### 1. 确认 sandbox 状态

先看当前是否启用了 sandbox：

```bash
openclaw sandbox status
```

正常输出应包含类似：

```text
fs.mode       = overlay
fs.denyDelete = true
cmd.sandbox   = on
trashDir      = ~/.openclaw/trash
```

如果 `fs.denyDelete` 为 `false`，后面配置不会生效。

### 2. 配置文件系统层拦截

在 `~/.openclaw/config.toml` 中配置：

```toml
[sandbox.fs]
mode = "overlay"                 # 写时复制，不直接改真实文件
denyDelete = true                # 拦截 unlink/rmdir
trashDir = "~/.openclaw/trash"   # 删除转为回收站
allowedWritePaths = [
  "./workspace",
  "./.cache",
  "./venv",
]
```

`denyDelete = true` 是关键。它不是在命令层拦 `rm`，而是在文件系统层拦截 `unlink`/`rmdir` 系统调用。即使 Agent 用 Python 脚本调用 `os.remove()`，也会收到 `EPERM`。

`allowedWritePaths` 是白名单，Agent 只能在这些路径下写文件。注意路径要写相对路径或绝对路径，不要只写 `workspace`，否则从其他工作目录启动时会匹配不到。

### 3. 命令层作为补充

命令沙箱只能作为第二层，不要单独依赖：

```toml
[sandbox.command]
allowlist = ["ls", "cat", "grep", "python", "node", "git"]
denyPatterns = ["rm\\s+(-rf?|/)"]
confirmOnDeny = true
```

这里的 `denyPatterns` 主要拦 shell 直接执行 `rm -rf`。但 Agent 可以通过生成脚本绕过命令正则，所以文件系统层才是主力。

### 4. 限制 MCP 工具权限

如果通过 MCP 暴露文件系统能力，需要单独限制：

```toml
[[mcp.servers]]
name = "filesystem"
allowedPaths = ["./workspace"]
denyDelete = true
```

MCP 服务器是独立进程，主沙箱不自动覆盖。很多误删来自 MCP 文件服务权限过大，这一层必须单独配置。

### 5. 测试拦截效果

用一个简单 prompt 测试：

```text
请删除 ./tmp/test.txt
```

观察日志 `~/.openclaw/logs/sandbox.log`，应看到类似：

```text
DENY unlink ./tmp/test.txt (sandbox.fs.denyDelete)
```

Agent 会收到 `EPERM` 或“操作被拒绝”的返回，而不是继续执行。如果没看到 `DENY` 日志，说明配置未生效或路径不在拦截范围内。

## 踩坑点

- **只配命令正则不够**。Agent 会写脚本直接调用 `os.remove` 或 `fs.unlink`，命令层拦不到。必须在文件系统层做 `denyDelete`。
- **overlay 模式下新文件“消失”**。Agent 写的新文件在 overlay 层，未提交前真实目录看不到。如果用户找不到文件，需要执行 `openclaw sandbox commit` 提交，或配置自动提交。
- **symlink 逃逸**。`allowedWritePaths` 里的路径如果包含软链接，可能指向外部目录。OpenClaw 会用 `realpath` 校验，但如果你的路径本身是软链接且未规范化，可能绕过。建议配置后测试 `ln -s /tmp ./workspace/link` 再让 Agent 写 `./workspace/link/test`。
- **MCP 服务器没单独限制**。主 sandbox 开了，但 MCP filesystem 服务器还开着全局读写，等于后门。每个 MCP 服务都要单独配置 `allowedPaths`。
- **trashDir 膨胀**。删除被转成回收站后会持续占用空间，建议定期清理 `~/.openclaw/trash`。
- **测试后忘记恢复**。有人为了测试关闭 sandbox，之后忘记开启，结果生产环境裸奔。建议用 `openclaw sandbox status` 纳入日常检查。

## 可复用建议

1. **最小化 `allowedWritePaths`**：只给 Agent 当前任务需要的写入目录，别直接给 `~`。
2. **启用 `denyDelete + trash`**：删除先入回收站，即使漏网也能找回。
3. **每层独立测试**：文件系统层、命令层、MCP 层分别验证，不要只测一处。
4. **自动化任务用独立用户或容器**：配合系统级权限隔离，减少 sandbox 配置遗漏的影响。
5. **定期审计 sandbox 日志**：关注 `DENY` 之外的 `ALLOW` 记录，看 Agent 实际访问了哪些路径。

## 总结

OpenClaw 的 sandbox 之所以能阻止 Agent 误删文件，不是因为它让模型更聪明，而是因为它把删除操作变成了一个需要显式授权、可拦截、可回滚的动作。文件系统层 `denyDelete` 是核心，命令层和 MCP 权限是补充。配置时注意白名单路径、symlink 逃逸和 MCP 独立权限，才能真正做到“Agent 不会误删文件”。

---

