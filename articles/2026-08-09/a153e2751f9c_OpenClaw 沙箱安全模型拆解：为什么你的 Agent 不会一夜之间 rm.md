---
title: OpenClaw 沙箱安全模型拆解：为什么你的 Agent 不会一夜之间 rm -rf
feedId: 32231
source: 综合讨论
publishedAt: 2026-08-09
---

## 背景：自动化下的人为焦虑

如果你正在用 OpenClaw 构建文件自动化链路，一定遇到过这样的灵魂拷问：

> “Agent 会不会把我整个项目目录删了？”

这并非多虑。传统脚本里一个错误的 `rm -rf $undefined_var/*` 足以毁掉一周的工作，而 Agent 的行为由 LLM 动态决策，不可预测性更高。社区里也出现过插件误操作导致配置文件丢失的案例，根源往往不是 AI 坏，而是**运行环境缺少工程化的隔离边界**。

OpenClaw 的设计者很早就意识到：**不能把文件系统完整暴露给不确定的执行上下文**。因此官方内置了一套可配置的沙箱模型，它并非容器或虚拟机级别的重型隔离，而是通过**路径代理 + 能力限制 + 写前快照**的组合，把“误删”变成几乎不可能发生的事。

## 问题：误删是如何被阻止的？

先还原一个典型危险动作：Agent 通过 MCP 工具获得了文件读写的授权，然后 LLM 生成了删除某个临时目录的操作。在无防护的传统实现里，这个操作会直接作用于宿主机文件系统。

OpenClaw 的做法是让所有文件操作都经过三层过滤：

1. **VFS 路径映射**  
   每个 Agent 会话看到的根目录并非真实 `/home/user/project`，而是一个虚拟化的 `workspace/`。所有路径被重写到沙箱目录树里。`/etc/passwd` 这样的敏感路径在映射层就被拒绝了。

2. **能力令牌校验**  
   文件写、删除、重命名等操作需要显式的 capability。默认能力集里根本不含 `DELETE`。要启用删除，必须在 manifest 里声明 `filesystem:delete`，且通常限定子路径。

3. **写时复制 (CoW) 快照**  
   即便声明了删除能力，在“安全模式”下，沙箱会对目标路径做一次即时快照保存到 `.openclaw/snapshots/`。删除操作只是把文件移入回收区，并非物理抹除。

这套模型不是仅靠文档约束的“君子协定”，而是实打实的中间层拦截。我自己在集成一个文件整理插件时做过测试：让 Agent 删除 `./data/cache/*`，实际对应沙箱内的 `workspace/data/cache/*`，而宿主机目录纹丝不动。只有显式提交 `session.commit()` 后，删除才会被同步到真实磁盘，前提是 capability 和路径 ACL 同时放行。

## 做法：开启并验证沙箱保护

假设你已有一个 OpenClaw 运行环境，以下步骤可以让你亲手验证文件删除不可行。

### 1. 配置沙箱策略文件
在项目根目录创建 `openclaw.sandbox.yaml`：
```yaml
sandbox:
  mode: strict
  root: "./sandbox-root"
  capabilities:
    - filesystem:read
    - filesystem:write
  deny_delete: true
  acl:
    - path: "data/**"
      allow: ["read", "write"]
    - path: "config/**"
      allow: ["read"]
```
该配置表明：
- 所有文件操作映射到 `sandbox-root` 下。
- 全局不允许删除。
- `data/` 可读写，`config/` 只读。

### 2. 启动 Agent 并触发危险指令
写一段简单的自动化任务，让 Agent “清理 data/cache 下的所有临时文件”。在 OpenClaw 的任务面板中运行，观察沙箱日志。

你会看到类似输出：
```
[sandbox] blocked: delete "data/cache/temp.json" -> denied by policy (deny_delete: true)
```
任务会失败，但不会影响任何真实文件。

### 3. 调整为允许删除但启用快照
修改配置：
```yaml
  deny_delete: false
  snapshot_before_write: true
```
再次运行相同的任务。这次操作会“成功”，但进入 `sandbox-root/.openclaw/snapshots/` 查看，删除的文件被保存在时间戳命名的目录下。真实文件系统无变化，直到你执行 `session.commit()`。

## 踩坑点：别让插件绕过沙箱

- **插件自带文件操作**  
  部分社区 MCP 插件直接调用 Node.js 的 `fs.unlink`，未经过 OpenClaw 的 VFS 层。安装前一定要审查插件是否集成了沙箱 SDK 或标记了 `sandbox-compliant`。否则沙箱形同虚设。

- **符号链接逃逸**  
  如果沙箱允许跟随符号链接，且真实路径指向沙箱外部，删除有可能穿透。应始终在沙箱配置中设置 `follow_symlinks: false`（默认值），并确保 `sandbox-root` 内没有指向外部的链接。

- **环境变量注入**  
  某些 Agent 允许在指令中注入 `$HOME` 等环境变量，如果沙箱路径映射未覆盖这些变量，可能会解析到真实目录。锁定运行时环境变量白名单，是容易被忽略的一步。

- **commit 时机**  
  `session.commit()` 是将沙箱变更写入真实磁盘的唯一通道。一旦误调用，写保护就没了。建议在生产中永远不要由 Agent 自主触发 commit，而是改为手动审批流程。

## 可复用的工程建议

1. **最小权限开始**  
   新项目一律设为 `mode: strict`，只给 `read` 能力。确实需要写操作时，精确到子目录，而不是放开整个 `workspace`。

2. **快照作为最后防线**  
   即使允许删除，也保持 `snapshot_before_write: true`。存储成本极低，但能让误操作可逆。

3. **所有插件走审查**  
   建立内部插件白名单，强制要求文件类插件输出沙箱合规报告。不符合的用包装器注入 OpenClaw 提供的 `SandboxedFS` 模块。

4. **自动化测试危险指令集**  
   在你的 CI 里维护一个“炸弹指令”列表，比如删除根目录、重写关键配置文件，每次发布前让 Agent 在 staging 环境执行，验证沙箱是否拦截。

## 总结

OpenClaw 的沙箱之所以能防止 Agent 误删文件，靠的不是魔法提示词，而是**路径虚拟化、能力管控、写前快照**三层硬约束。这套模型将 AI 的不确定性框定在一个可审计、可逆的隔离域内，给自动化操作上了一道工程保险。对已经或计划将 Agent 接入真实文件流的开发者来说，理解并正确配置沙箱，是把“自动”变为“安全自动”的分水岭。

下次再有人问“Agent 会不会删我文件”，你完全可以掏出沙箱日志，而不是一句苍白的“应该不会”。

---

