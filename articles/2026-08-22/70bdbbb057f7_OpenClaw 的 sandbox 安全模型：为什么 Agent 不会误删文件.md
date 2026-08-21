---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 34111
source: 综合讨论
publishedAt: 2026-08-22
---

## 背景

在 OpenClaw 里跑自动化任务时，最常见的事故不是 Agent 不够聪明，而是它“太听话”。一句 `rm -rf ./cache` 如果拼接错路径，或者某个 MCP 插件返回了错误的文件列表，就可能把宿主目录里的配置、照片、项目源码直接带走。

传统 Agent 框架默认继承当前用户的完整文件权限，这等于把所有操作的“后悔药”收走了。OpenClaw 的 sandbox 安全模型做的事情很直接：把文件系统的读、写、删拆开，默认不让你碰宿主文件，只有显式允许的目录才可以写入，删除操作默认进回收站而不是直接 unlink。

## 问题

一个典型的误删链路是这样的：

1. Agent 需要清理临时文件；
2. 它根据某个工具返回的路径拼出删除命令；
3. 路径里多了一个 `../`，或者工具返回了绝对路径；
4. 命令执行，宿主的 `~/Documents` 被删；
5. 没有审计、没有备份，不可恢复。

OpenClaw 的 sandbox 模型从三个层面解决这个问题：**隔离执行环境、限制写权限、将删除变为可逆操作**。即使 Agent 产生错误意图，也很难直接触达重要文件。

## 做法 / 步骤

我现在的 OpenClaw 配置里，sandbox 部分大致长这样：

```yaml
sandbox:
  enabled: true
  fs:
    backend: overlay
    read_only: true
    writable:
      - ~/.openclaw/workspace
      - /tmp/openclaw
    deny_delete: true
    trash_path: ~/.openclaw/trash
    allow_overwrite: false
  commands:
    blocked:
      - "rm -rf /"
      - "mkfs"
      - "dd if="
    allow_list: false
  audit:
    enabled: true
    log_path: ~/.openclaw/logs/fs_audit.jsonl
```

核心步骤是：

1. **开启 overlay 文件系统**  
   宿主目录只读挂载，Agent 的写操作被重定向到 overlay 层。即使它修改了 `/etc/hosts`，也只是在沙箱层改，宿主不受影响。

2. **只开放必要可写目录**  
   我的自动化任务只会写 `~/.openclaw/workspace`。其他路径一律拒绝写入。

3. **删除转回收站**  
   `deny_delete: true` 配合 `trash_path`，让 `rm` 变成 `move to trash`。误删后可以手动恢复。

4. **命令级拦截**  
   对 `rm -rf /`、`mkfs` 这类高危命令直接阻断。如果不想维护黑名单，可以改成 `allow_list: true`，只允许 Agent 执行白名单里的命令。

5. **审计日志**  
   每次写操作会记录真实路径、命令、调用来源。出问题时，可以从日志反推 Agent 到底做了什么。

## 踩坑点

实际用下来有几个容易忽略的地方：

- **路径映射不一致**  
  沙箱内 Agent 看到的路径可能是 `/workspace`，但脚本里写的是 `~/workspace`。这会导致任务失败。建议在任务开始时统一注入 `HOME=/workspace`，或者在配置里做路径映射。

- **白名单过宽**  
  有人图省事把整个 `$HOME` 加进 `writable`，那 sandbox 基本等于没开。最好按任务单独建临时目录，任务结束后回收。

- **只防删除不防覆盖**  
  `deny_delete` 只能拦住删除，如果 Agent 用 `>` 或 `cp` 覆盖了重要文件，一样会丢数据。所以要开启 `allow_overwrite: false`，或者对重要目录单独设置写保护。

- **外部副作用管不到**  
  沙箱只能约束本地文件系统。如果 Agent 通过 MCP 插件调了云盘 API、数据库删除接口，本地 sandbox 是拦不住的。这类操作要在插件层做权限控制。

- **性能开销**  
  overlay 模式对大文件频繁写不太友好。如果任务需要处理大量媒体文件，可以只对高风险任务开 overlay，其他任务用普通目录，依赖审计兜底。

## 可复用建议

1. **最小权限原则**  
   每次任务只给一个临时目录，不要复用全局 workspace。

2. **删除转回收站**  
   尽量不让 Agent 直接执行 `rm`。在 sandbox 层把删除重定向到 trash，定期清理 trash 即可。

3. **先 dry-run 再执行**  
   OpenClaw 支持 plan 模式，先让 Agent 输出计划，人工确认后再执行。对批量删除、批量重命名这类任务特别有用。

4. **MCP 插件单独授权**  
   不要给插件继承全局文件权限。每个插件单独声明它需要的路径和操作类型。

5. **定期回看审计日志**  
   审计日志不是摆设。每周看一眼 `fs_audit.jsonl`，能发现很多“差点出事”的操作。

## 总结

OpenClaw 的 sandbox 安全模型不是让 Agent 变得绝对安全，而是把误删、误改的风险限制在可恢复范围内。它的价值在于：默认拒绝、删除可逆、操作可审计。对于跑自动化、MCP 插件、批量文件处理的用户来说，这套机制比“相信 Agent 不会犯错”现实得多。

真正工程化的做法是：sandbox 打底，策略和审计兜底，外加任务前的 dry-run 和任务后的快照对比。不要指望单一机制解决所有问题，分层防护才是关键。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/1b8cf8bc09b5fd51.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/cf528d17263a4fae.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/5aab2ec07674307d.png)

