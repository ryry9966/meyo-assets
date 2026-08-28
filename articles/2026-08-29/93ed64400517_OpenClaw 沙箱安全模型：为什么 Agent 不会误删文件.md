---
title: OpenClaw 沙箱安全模型：为什么 Agent 不会误删文件
feedId: 35128
source: 综合讨论
publishedAt: 2026-08-29
---

## 背景

在 OpenClaw 里跑 Agent、MCP 插件或自动化任务时，最让人犹豫的往往不是模型会不会“乱想”，而是它执行工具时会不会把宿主环境弄坏。尤其是文件删除：`rm -rf`、清理临时目录、批量重命名，一旦越界就很难恢复。

OpenClaw 的沙箱安全模型把“Agent 提出动作”和“动作真正落到文件系统”拆开。默认情况下，Agent 并不直接拥有宿主文件系统的删除权限，而是经过一层边界和策略代理。这里说的不是模型学不会删除，而是基础设施不让它直接删。

## 问题拆解

一次误删通常需要两个条件同时成立：

1. 删除目标位于可写挂载范围内，或路径解析后落到了可写区域；
2. 删除操作没有经过确认、拦截或可恢复处理。

OpenClaw 的做法是同时破坏这两个条件：缩小可写面，把删除变成可拦截、可回滚的动作。

## 做法/步骤

### 1. 文件系统边界最小化

- 根文件系统默认只读；
- 只允许写 `/workspace` 或显式挂载的数据卷；
- 宿主 `$HOME`、`/etc`、`/usr` 等不与 Agent 共享。

### 2. 删除操作改造成“暂存删除”

不是直接 `unlink`，而是将目标移动到 `.trash/` 或快照目录，并设置 TTL。这样“误删”先变成“暂时消失”，窗口期内可恢复。

### 3. 危险动作走审批流

对 `rm`、`unlink`、`delete`、`rename` 到可写区外等操作，策略引擎先阻断，要求 human-in-the-loop。批处理任务可对高置信度操作自动批准，但默认不静默放行。

### 4. 路径解析强制 realpath

执行前对目标路径做 `realpath`，拒绝符号链接逃逸。比如 Agent 试图删除 `/workspace/link`，如果 link 指向 `/home/user/important`，直接拒绝。

### 5. 变更前 dry-run

让 Agent 输出 planned changes，策略层根据目标路径、命令类型、大小、扩展名等判断。dry-run 通过后才真正执行。

### 6. 快照/回滚

在可写层使用 overlay、btrfs snapshots 或 session 级备份。误删后可以回滚最近一次快照。

概念性策略示例：

```yaml
sandbox:
  rootfs: read-only
  writable:
    - /workspace
  delete_policy: trash
  trash_ttl_hours: 72
  approvals:
    - pattern: "rm|unlink|delete"
      require: human
  symlink_check: realpath
```

不同版本字段可能不同，但核心是这六项。

## 踩坑点

- **挂载过大**：把宿主 `/home` 直接挂给 Agent，等于把沙箱边界做没了。宁可多开一个 `/workspace`。
- **只做前缀匹配不做 realpath**：`/workspace/link` 可能指向外部，必须解析真实路径。
- **只拦 `rm` 不拦 `rename`/`mv`**：删除可以通过 rename 到可写范围外实现，策略要覆盖 move/overwrite。
- **trash 策略占空间**：大量批量任务会让 `.trash` 膨胀，需要 TTL 和定期清理，否则磁盘会被“软删除”占满。
- **快照不是默认生效**：在 ext4 上 overlay 需要明确配置；不要假设沙箱自带完整回滚。
- **审批流太松或太紧**：太松等于没拦，太紧会拖垮自动化体验。建议按命令类别和路径分级，而不是全局人工确认。

## 可复用建议

- 默认拒绝，白名单放行。Agent 只读系统，只写 `/workspace`。
- 删除先 trash，再定期清理；对重要目录额外要求 human 确认。
- 所有文件变更记录审计日志：`action, target, resolved_path, result, ts`。出问题能还原链路。
- 定期做恢复演练：从 trash 恢复、从快照回滚，确认流程可用。
- 插件/MCP 工具也要走同一套文件访问代理，不要给个别插件开特权。
- 把“删除”当危险动作，而不是把“模型”当安全边界。

## 总结

OpenClaw 沙箱能让 Agent 不误删文件，不是因为模型格外谨慎，而是因为权限边界、路径解析、删除代理、审批和回滚形成了纵深防御。Agent 可以表达删除意图，但真正落到 `unlink` 之前，必须经过明确的策略关口。对工程实践来说，这比“模型懂文件安全”可靠得多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/691130dafee24cdf.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/dde0aaa7ba614192.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/56edbf51f1046904.png)

