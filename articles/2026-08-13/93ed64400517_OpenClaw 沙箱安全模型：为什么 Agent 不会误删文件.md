---
title: OpenClaw 沙箱安全模型：为什么 Agent 不会误删文件
feedId: 32899
source: 综合讨论
publishedAt: 2026-08-13
---

# OpenClaw 沙箱安全模型：为什么 Agent 不会误删文件

## 背景

用 Agent 做文件整理、批量重命名、日志清理时，最怕的不是它做得慢，而是它“手快”：一个错误的 `rm -rf` 或路径拼写错误，可能直接把宿主目录带走。传统做法是在 prompt 里反复强调“不要删除重要文件”“操作前先确认”，但大模型在长任务中仍然可能忽略约束，尤其是工具链复杂、上下文变长之后。

OpenClaw 的沙箱模型不是简单地“禁止删除”，而是把文件操作放进一个可控边界内：Agent 只能看到被允许的目录，写操作默认落盘到安全层，删除动作被拦截或转成可回滚操作。这样即使 Agent 出现误判，也不会直接破坏宿主的真实文件。

## 问题

Agent 的文件操作风险主要来自三点：

1. **路径越界**：工具调用时生成了 `../../` 或绝对路径。
2. **破坏性命令**：`rm`、`unlink`、`truncate` 等操作直接作用在真实文件上。
3. **权限过大**：MCP 文件服务或插件把整个 `$HOME` 暴露给 Agent。

因此，安全模型要解决的不是“让 Agent 变聪明”，而是“让错误没有机会落到真实文件上”。

## 做法/步骤

### 1. 限制可写工作区

在 OpenClaw 的配置中，把 Agent 的文件工具默认根目录设为会话级工作区：

```yaml
sandbox:
  fs:
    mode: workspace
    root: ~/.openclaw/workspace/{session_id}
    readonly_host: true
```

这样 Agent 的 `read`、`write`、`list` 都只能发生在 `~/.openclaw/workspace/<session_id>` 下。宿主目录默认只读，即使工具调用带了 `~/Documents` 也会被拒绝。

### 2. 删除操作转回收站

不要直接开放 `rm` 工具。可以在工具层加一个 `safe_delete` wrapper：内部调用 `trash-put` 或 `mv` 到回收站目录，而不是直接 `unlink`。配置示例：

```yaml
tools:
  file:
    delete_policy: trash
    trash_dir: ~/.openclaw/trash
```

这样即使 Agent 误删，文件也只是被移动到了回收站，可以恢复。

### 3. 启用 overlay 写时复制

更稳妥的做法是 overlay 模式。Agent 看到的文件系统是“宿主目录 + 可写覆盖层”：

- 读操作：优先读覆盖层，没有则读宿主只读层。
- 写操作：全部写入覆盖层，不触碰宿主文件。
- 删除操作：在覆盖层中标记为“白障”，宿主文件仍在。

配置：

```yaml
sandbox:
  fs:
    mode: overlay
    lower: ~/projects
    upper: ~/.openclaw/overlay/{session_id}
```

任务结束后，可以 `openclaw sandbox commit` 合并写操作，或 `openclaw sandbox rollback` 丢弃整个会话的修改。

### 4. MCP 工具权限最小化

很多文件误删来自 MCP 文件服务权限过大。OpenClaw 加载 MCP 插件时，应检查 manifest 中的 `permissions`：

- 只读工具：`fs.read`、`fs.list`、`fs.stat`
- 写工具：`fs.write`、`fs.mkdir`
- 删除工具：默认不加载；确需使用时再单独开 `fs.delete`，并加上确认门槛

例如：

```yaml
mcp:
  filesystem:
    allowed_tools: [read, write, list, mkdir]
    deny_tools: [delete, rmdir, truncate]
```

### 5. 审计日志与回滚

每次文件写操作都记录 journal：

```json
{"op":"write","path":"workspace/a.txt","before":"hash1","after":"hash2","ts":"..."}
{"op":"delete","path":"workspace/b.txt","moved_to":"trash/b.txt","ts":"..."}
```

可以用 `openclaw sandbox diff --session <id>` 查看本次 Agent 到底改了什么。出问题时，按 journal 反向操作即可恢复。

## 踩坑点

1. **不要只靠 prompt 约束**。模型在复杂任务中会绕过，尤其是多轮工具调用后，prompt 中的“不要删除”很容易被遗忘。沙箱必须做在工具/系统层。

2. **回收站路径不能在沙箱内**。如果 `trash_dir` 被 Agent 可写，它可能会把回收站也清空。回收站应放在沙箱边界之外，且只允许 `safe_delete` 写入。

3. **符号链接逃逸**。即使限制了根目录，Agent 可能通过写入符号链接指向宿主文件来绕过。沙箱需要禁止跟随 symlink，或对每个路径做 `realpath` 校验，确保真实路径仍在工作区。

4. **overlay 模式的性能**。大文件或大量小文件场景下，overlay 层会占用额外磁盘，写入性能也会下降。建议只对任务涉及的小范围目录启用 overlay，大文件目录用“只读 + 回收站”即可。

5. **不要全局禁用删除**。如果完全禁止删除，自动化清理类任务会大量失败，反而降低 Agent 可用性。更好的做法是：默认转回收站，关键路径需要显式确认或白名单。

## 可复用建议

- **默认 deny，显式 allow**：文件工具初始只读，写和删除单独开启。
- **分环境策略**：开发环境用 overlay 方便试错；生产环境只读 + 审批。
- **统一入口**：所有文件操作走同一个 wrapper，避免 Agent 绕过安全层直接调用底层命令。
- **定期清理**：overlay 层和回收站要定期清理，否则磁盘会被历史会话占满。
- **用 git 兜底**：如果项目本身是 git 仓库，任务前打 tag 或 stash，是最容易实现的版本回滚。

## 总结

OpenClaw 的沙箱安全模型核心思路是：**不给 Agent 直接触碰真实文件的机会**。通过工作区隔离、删除转回收站、overlay 写时复制、MCP 权限最小化和审计日志，把“误删”从不可逆事故变成可回滚操作。工程上不要指望模型自己小心，而是把安全性做成默认配置。这样 Agent 才能真正用于文件自动化，而不是每次跑完都要胆战心惊地检查宿主目录。

---

