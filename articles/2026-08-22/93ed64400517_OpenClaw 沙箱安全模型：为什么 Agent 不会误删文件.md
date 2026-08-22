---
title: OpenClaw 沙箱安全模型：为什么 Agent 不会误删文件
feedId: 34156
source: 综合讨论
publishedAt: 2026-08-22
---

## 背景

在 OpenClaw 里，Agent 不只做对话，还会调 shell、跑脚本、通过 MCP 或插件操作文件。文件删除是最容易翻车的一类操作：路径拼接错、清理逻辑误判、插件带删除能力但没被约束，都可能让一个自动化任务把项目目录清空。

只靠 prompt 说“不要乱删”不可靠。真正能兜底的是 sandbox 执行层：让 Agent 根本没有能力删除边界外的文件。

## 问题

常见的误删路径不是 Agent 主动作恶，而是工程上的漏洞：

- `rm -rf $PATH` 里 `$PATH` 为空，变成删当前目录；
- Python 脚本调用 `shutil.rmtree`，shell 命令白名单完全拦不到；
- MCP 工具暴露了 `filesystem:delete`，但用户不知道；
- 自动化清理逻辑在错误分支执行，删掉生成物时把源文件一起删了。

这些场景都说明：安全边界不能放在自然语言层，必须放在文件系统、syscall 和权限层。

## 做法 / 步骤

**1. 划出工作区边界**

在 OpenClaw 的 sandbox profile 里，不要挂载整个家目录。只把当前任务目录挂到 `/workspace`，系统目录只读或完全不可见。

```yaml
sandbox:
  engine: bubblewrap
  rootfs: read-only
  workspace: /workspace
  bind:
    - /home/user/projects/current-agent-task:/workspace
```

字段名按实际版本调整，核心思路是：Agent 能写的只有 `/workspace`。

**2. 删除操作转回收站**

优先把 `rm` 之类的命令 alias 到 `trash-put`。更可靠的是在 sandbox 内拦截 `unlink/unlinkat` 等 syscall，只允许它们在 `/workspace/.trash` 范围内生效。这样即使 Agent 用 Python `os.remove`，文件也只是进回收站。

**3. 危险操作审批**

对 `rm -rf`、`find -delete`、`truncate`、`mv -f` 覆盖等操作进入审批队列。审批按风险分级，不要全量审批，否则用户会直接关掉审批。

**4. 操作前快照**

任务开始前创建 overlay 快照或 ZFS snapshot，删除发生后可以回滚。保留最近 5-10 个快照，避免空间膨胀。

**5. 插件 / MCP 权限分级**

插件需要声明 `filesystem:write` 和 `filesystem:delete`，其中 `delete` 默认拒绝。只有用户显式授权后，该插件才可能删除文件。

**6. 审计日志**

记录每个删除动作的来源 action id、插件、原始命令和 syscall。这样能定位到底是哪个 Agent 步骤触发了删除，而不是只看到一条 `rm` 日志。

## 踩坑点

- **只拦 `rm` 不够。** 真实踩过：Agent 用 Python 脚本执行 `shutil.rmtree`，shell 层白名单完全看不到。所以必须在 syscall 或文件系统权限层做限制。
- **不要把 `~` 挂进沙箱。** `rm -rf ~/.cache` 一旦执行，家目录配置可能被清空。只挂任务目录。
- **回收站不清理会膨胀。** 设置保留时间，比如 7 天或 500MB 上限。
- **审批疲劳。** 全量审批会逼用户关闭审批。只对删除、覆盖、系统路径审批。
- **快照不是备份。** 频繁写大文件时 overlay 快照会占用大量空间，建议数据目录用 COW 文件系统，或限制快照大小。
- **插件可能绕过 shell 直接删除。** 命令白名单只能作为第一层，最终需要文件系统边界兜底。

## 可复用建议

- 默认 deny：新插件、新 MCP server 默认没有 `filesystem:delete`。
- 工作目录和家目录分离，数据目录只读挂载。
- 删除优先走 trash，而不是直接 unlink。
- 危险写操作前自动快照，保留最近 N 个。
- 审计日志与 agent action id 关联，不要只记命令。
- 上线自动化任务前先 dry-run，打印将删除的路径，确认后再执行。

## 总结

Agent 不会误删文件，不是因为它被 prompt 教育得足够好，而是因为沙箱把删除能力限制在工作区内，并配合回收站、审批、快照和审计。安全边界应该放在执行层，而不是自然语言层。只有默认拒绝、最小权限和可回滚同时存在，自动化文件操作才值得信任。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/a639f45b89c5ba05.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/0720aa797020f78d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/d355ffa1c184ce90.png)

