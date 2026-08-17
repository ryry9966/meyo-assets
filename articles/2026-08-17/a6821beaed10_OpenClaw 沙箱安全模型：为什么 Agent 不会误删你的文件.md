---
title: OpenClaw 沙箱安全模型：为什么 Agent 不会误删你的文件
feedId: 33547
source: 综合讨论
publishedAt: 2026-08-17
---

## 背景

给 Agent 接文件系统工具后，最让人不安的不是它读错文件，而是它在错误的时间对错误的路径执行了 `delete`、`rm` 或 `move`。很多自动化事故不是模型“变坏”，而是我们把宿主机的文件权限几乎原样交给了 Agent。

OpenClaw 的 sandbox 不是给 Agent 加一个“安全提示词”，而是在工具执行层做路径隔离、能力约束和可恢复删除。理解这套模型后，Agent 误删文件的主要路径会被堵住。

## 问题

直接暴露文件系统时，常见风险集中在三类：

1. **路径不可控**：Agent 生成了 `../`、绝对路径或拼接错误，实际操作落在工作区之外。
2. **删除即物理删除**：`rm`、`unlink` 执行后没有中间态，误删后只能靠外部备份。
3. **工具权限过大**：一个 `shell` 工具或 MCP 插件可以绕过所有文件 API 限制，直接操作原生文件系统。

这些问题的共同点不是模型能力不够，而是安全边界没有落在执行层。

## 做法 / 步骤

在 OpenClaw 中，我会按下面顺序配置 sandbox，核心思路是：先隔离，再限制能力，最后保证可恢复。

### 1. 固定工作区根目录

先把 Agent 能接触的路径收窄到一个 workspace：

```yaml
sandbox:
  root: .openclaw/workspace
  defaultPolicy: read-only
```

所有相对路径先拼到 `root` 下，不允许直接使用绝对路径。这样即使 Agent 输出 `/etc/passwd`，也不会落到宿主机的 `/etc`。

### 2. 写操作单独授权

默认只读，写、删除、重命名等操作需要显式声明：

```yaml
capabilities:
  fs.write: true
  fs.delete: false
  fs.move: true
```

我更倾向把 `fs.delete` 默认设为 `false`，只有在批处理任务中临时打开。Agent 如果需要清理临时文件，走专门的 `trash` 工具，而不是通用删除。

### 3. 删除改成进入回收站

关键一步：不让 Agent 直接调用物理删除。把删除实现改为移动到 staging 目录：

```text
.openclaw/trash/2025-01-01T101500Z/
```

这样即使删错，也能从回收站恢复。定期清理回收站由用户或独立任务执行，Agent 不参与。

### 4. 路径校验用 realpath

路径校验不能只做字符串前缀匹配。必须解析符号链接后再判断：

```text
用户命令 -> 拼接绝对路径 -> realpath -> 检查是否在 workspace 内 -> 放行/拒绝
```

否则沙箱内一个指向外部目录的 symlink 就能实现逃逸。

### 5. 危险操作先 dry-run

对批量删除、移动、覆盖类操作，第一轮只生成变更清单：

```text
dry-run:
  action: delete
  candidates:
    - .openclaw/workspace/tmp/a.log
    - .openclaw/workspace/cache/b.tmp
```

用户确认后，第二轮才真正执行。模型可以参与生成候选列表，但确认权留在人这边。

### 6. 保留审计日志

每次文件操作记录五元组：

```text
command, cwd, resolvedPath, decision, timestamp
```

日志单独存放，Agent 只读不可写。出问题时先看日志，而不是先怀疑模型。

## 踩坑点

实际落地时，有几个地方容易翻车：

- **symlink 逃逸**：只做字符串前缀检查不够，必须 `realpath` 后再判断。否则 workspace 内一个软链指向 `~/.ssh`，前缀检查会认为安全。
- **MCP 插件绕过沙箱**：如果 MCP server 自己用原生 `fs` 操作文件，OpenClaw 外层的 sandbox 管不到。需要让 MCP server 也运行在受限目录，或者限制它只能访问指定挂载点。
- **shell 工具比文件工具危险**：给 Agent 一个 `shell` 工具后，`rm -rf`、`find -delete` 都可能绕过文件 API 层。shell 工具要么不给，要么放在独立容器/沙箱里。
- **trash 跨文件系统失败**：把文件从 workspace 移动到 trash 目录时，如果两者不在同一个文件系统，会触发 `EXDEV`。trash 目录最好建在 workspace 同一挂载点下。
- **白名单过严导致任务失败**：路径卡得太死，Agent 频繁报错，用户可能一怒之下放开全部权限。安全策略要在可用性和约束之间找平衡，不要搞成“要么锁死，要么全开”。

## 可复用建议

1. **默认只读**：写操作单独工具，删除默认关闭。
2. **删除走 trash**：物理删除只保留给回收站清理任务。
3. **路径校验必须 realpath**：并且要求最终路径在 workspace 前缀内。
4. **MCP 插件与 Agent 共用同一 sandbox 边界**：不要把插件当“可信组件”。
5. **开启决策日志**：至少保留 24 小时，方便复盘。
6. **重要目录用 deny 覆盖 allow**：如 `~/.ssh`、`~/.aws`、`.git` 等，即使白名单误配，也要显式拒绝。

## 总结

OpenClaw 的 sandbox 安全模型不是让 Agent 变聪明，而是让它“想删也删不到”。通过 workspace 隔离、写操作授权、删除转回收站、realpath 校验和审计日志，误删文件的主要通道被关闭。

它仍然不是绝对安全，尤其当你主动给 shell 或让 MCP 插件绕过边界时。但在默认只读、删除可恢复、路径受约束的配置下，Agent 误删重要文件会从一个高频事故变成需要连续多个环节同时失效才会发生的小概率事件。

安全的目标不是完全不出错，而是把爆炸半径控制在你愿意承担的范围内。

---

