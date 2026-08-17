---
title: OpenClaw sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 33649
source: 综合讨论
publishedAt: 2026-08-18
---

## 背景

给 OpenClaw 接入 MCP filesystem 或文件类插件后，最让人紧张的不是 Agent 能力不够，而是它“太敢做”。一旦模型误读指令，或者工具描述里出现 `delete_file`、`rm`、`truncate`，很多人会担心项目目录被一键清空。

单纯靠 prompt 约束“不要删除重要文件”并不可靠。模型在长上下文、工具调用链和自动化任务里，很容易把某个临时目录、缓存目录或文件当作可清理对象。因此，OpenClaw 的 sandbox 模型重点不是“相信模型”，而是让破坏性操作在到达宿主文件系统之前，先经过边界、审批和回滚机制。

## 问题

直接暴露宿主文件系统时，主要有三个风险：

1. **路径穿越**：工具只做字符串拼接，`../../` 或符号链接可能跳出预期目录。
2. **不可逆删除**：`rm -rf`、覆盖写、移动文件一旦执行，很难恢复。
3. **无审计**：Agent 执行了什么、拒绝了什么，没有记录，出问题只能靠猜。

OpenClaw 的 sandbox 安全模型要解决的就是这三件事：**边界隔离、破坏操作受控、全程可审计**。

## 做法/步骤

以一次实际配置为例。假设 OpenClaw 需要操作某个项目数据目录，但我不希望它碰主目录、配置目录或系统目录。

### 1. 设置 sandbox 根目录

不要让 Agent 的默认工作目录直接落在用户主目录。把根目录限制在一个专用目录：

```yaml
# 示意配置，字段名以你的 OpenClaw 版本为准
sandbox:
  root: ~/.openclaw/sandbox
  deny_path_escape: true
  resolve_symlinks: true
```

关键不是设置 root，而是 `deny_path_escape` 和 `resolve_symlinks`。路径必须先解析真实路径，再判断是否仍在 sandbox root 内。否则符号链接可以轻松绕过。

### 2. 限制 MCP filesystem 的可访问目录

如果你使用 MCP filesystem server，不要给 `/` 或整个用户目录。只给必要目录：

```yaml
mcp_filesystem:
  allowed_dirs:
    - ~/projects/app-data
    - ~/downloads/inbox
  write_enabled: false
```

先默认只读。等 Agent 确实需要写入时，再对单独目录开启写权限。

### 3. 删除操作进回收站，而不是物理删除

这是“不会误删文件”最核心的一点。OpenClaw 里文件删除的默认策略可以是移入回收站：

```yaml
filesystem:
  delete_mode: trash
  trash_dir: ~/.openclaw/trash
  destructive_require_confirmation: true
```

让 `delete_file` 工具在内部执行 `move to trash`，同时保留原路径元信息，方便恢复。物理删除需要显式开启 `destructive` 权限，并且通常还需要用户确认。

### 4. 先 dry-run，再执行

对批量删除、清理任务，先让工具返回“将要删除哪些路径”，由用户确认后再执行。这样即使模型判断错误，也只会在 dry-run 阶段暴露。

### 5. 检查审计日志

配置完成后，做一次故意越界测试：让 Agent 删除 `~/.openclaw/sandbox/../important.txt`。正常配置下，工具应该拒绝，并在审计日志里记录一次 `denied: path_escape`。

## 踩坑点

1. **只靠 prompt 不够**。不要写“你绝对不能删除文件”这种长约束，模型会在复杂任务中忽略。真正的限制必须落在工具实现层。
2. **sandbox root 不可与用户主目录重叠**。如果 root 是 `~`，那么内部文件都在允许范围内，边界形同虚设。
3. **符号链接很危险**。`~/.openclaw/sandbox/link -> /etc` 这种链接如果没解析，后续写入或删除都会穿透沙箱。
4. **回收站会占磁盘**。大量自动化任务会产生很多“已删除”文件，需要定期清理 `~/.openclaw/trash`，否则可能填满磁盘。
5. **dry-run 不等于审批**。有些 Agent 会返回“已删除”但实际未执行，或者工具描述里有 dry-run 字段但默认是 false。要看审计日志，不要只看模型输出。

## 可复用建议

- **最小权限原则**：MCP 或插件只给数据目录，不要给系统目录。
- **写入和删除分离**：写操作可以放开，但删除操作保持回收站模式，不要全局开启物理删除。
- **符号链接保护**：工具实现里先 `realpath`，再判断前缀。
- **自动化任务使用专用目录**：不要让定时任务直接操作项目目录，先写到一个 staging 目录，确认后再同步。
- **把审计日志纳入监控**：定期检查 `denied` 记录，异常拒绝可能代表模型行为偏离。

## 总结

OpenClaw 的 sandbox 安全模型并不是承诺“永不出错”，而是把误操作变成**可拦截、可回滚、可审计**的过程。文件默认进入回收站，破坏性操作需要显式开启，路径必须经过规范化和边界检查。这样即使模型在某一轮推理中犯错，也不会直接对宿主文件系统造成不可逆影响。

真正的安全感不来自“Agent 很聪明”，而来自**它没有直接触碰重要文件的权限**。

---

