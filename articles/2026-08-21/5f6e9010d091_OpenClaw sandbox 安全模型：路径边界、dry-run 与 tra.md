---
title: OpenClaw sandbox 安全模型：路径边界、dry-run 与 trash 如何拦住误删
feedId: 34062
source: 综合讨论
publishedAt: 2026-08-21
---

# OpenClaw sandbox 安全模型：路径边界、dry-run 与 trash 如何拦住误删

## 背景

Agent 现在不只是聊天，越来越多被接到文件整理、代码生成、数据处理、自动发布流程里。OpenClaw 的工具调用链中，文件系统操作是最高频的一类。模型生成路径时常见的错误包括：把相对路径写成绝对路径、把清理目录写成当前目录、变量拼接少了一层子目录。传统做法如果直接给 Agent 一个 shell，`rm -rf ./` 就可能把整个项目目录甚至 `$HOME` 下部分内容带走。

## 问题

误删之所以会发生，通常不是模型“想删”，而是三个原因叠加：路径歧义、参数注入、工具链缺少边界。比如一个任务要清理 `./cache`，模型输出 `rm -rf /cache` 或者 `rm -rf ./`，如果执行层没有约束，就出事故。OpenClaw 的 sandbox 不是简单用容器把进程隔离，而是在文件操作 API 层加了一层策略：工作区边界、写前预览、删除可回收、审计日志。

## 做法/步骤

OpenClaw 中一个最小可用的 sandbox 配置大致如下：

```yaml
sandbox:
  root: ./workspace
  readonly_mounts:
    - $HOME/datasets
  deny_patterns:
    - "**/.git/**"
    - "**/*.env"
    - "**/node_modules/**"
  blocked_commands:
    - "rm -rf /"
    - "mkfs"
    - "dd if=/dev/zero"
  trash: true
  dry_run: true
```

真正起作用的是执行链上的四步：

1. **路径规范化**：所有文件工具调用先做 `realpath`，拒绝 symlink 逃逸。任何最终路径必须落在 `root` 内，否则直接报错。
2. **写前 dry-run**：删除、覆盖、移动操作先返回一个 diff/patch 预览，列出会动到哪些文件、删除多少条路径。用户或策略匹配后才进入真实执行。
3. **trash 回收**：删除动作不是直接 `unlink`，而是移动到 `.openclaw/trash/<timestamp>/`，文件保留可恢复。`trash: true` 是默认建议。
4. **审计日志**：每次工具调用记录参数、归一化后的路径、dry-run 结果、最终动作，便于事后回滚。

例如，模型请求 `delete ./cache` 时，dry-run 会显示：

```
[preview] delete "workspace/cache/" (12 files, 4.3MB)
→ will move to .openclaw/trash/20250612-091532/
```

如果显示的是 `workspace/` 而不是 `workspace/cache/`，人工审批就能拦下来。

## 踩坑点

实践中这些地方容易出问题：

- **symlink 逃逸**：只做字符串前缀匹配不够。`workspace/link -> /etc` 如果不拒绝，写 `workspace/link/passwd` 会直接打到系统文件。必须在 realpath 后再做边界判断。
- **只拦 `rm` 是自欺欺人**：`find . -delete`、Python 的 `shutil.rmtree`、`unlink` 都能绕过命令黑名单。所以 OpenClaw 沙箱要作用在文件工具实现层，而不是 shell 命令黑名单。
- **dry-run 与真实执行不一致**：预览和实际执行之间如果工作区状态变了，可能删错。需要把 dry-run 的参数 hash 绑定到批准动作，或者对执行前再做一次路径校验。
- **把重要文件放 workspace**：沙箱边界不能替你判断文件重要性。`workspace/` 下的论文草稿也可能被当成缓存删掉。trash 不能关，审批不能省。

## 可复用建议

- 目录分层：`input/` 只读挂载，`output/` 可写，`tmp/` 可写并定期清理。不要让 Agent 整个 `$HOME` 可见。
- 对 MCP/插件同样限权：插件请求文件或数据库时，给它单独的 scoped token，不要复用主 Agent 的完整权限。
- 删除先 trash，定期清理 trash 时再走一次人工确认或策略白名单。
- 把所有文件写操作放进审计日志，至少保留最近 30 天，方便定位是哪次调用误删。

## 总结

OpenClaw 的 sandbox 安全模型并不能保证“绝对不误删”，它做的是把误删概率和爆炸半径降下来：边界隔离让灾难范围可控，dry-run 让删除意图可见，trash 让删除可恢复，审计让事故可追溯。把这四层配好，Agent 才适合从演示环境走进真实工作区。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/bf302b22ee260ad3.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/2d712f243c496fc8.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/b9c4d1554eedbde1.png)

