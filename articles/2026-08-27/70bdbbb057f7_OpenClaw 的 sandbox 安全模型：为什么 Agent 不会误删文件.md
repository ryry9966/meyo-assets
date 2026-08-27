---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 34950
source: 综合讨论
publishedAt: 2026-08-27
---

## 背景

OpenClaw 作为面向自动化的 Agent 框架，允许模型通过工具调用执行文件操作。但“能删文件”和“会误删文件”是两回事。很多用户第一次把 OpenClaw 接进项目时，最担心的不是它不够聪明，而是它会不会在某个提示词诱导或路径拼接错误下，把宿主机目录清空。

OpenClaw 的答案不是“加个确认弹窗”，而是一套分层 sandbox：默认把 Agent 关在工作区里，对文件系统做只读挂载、命令白名单、删除确认和审计。它的目标不是让 Agent 拥有 root 权限但“小心一点”，而是让 Agent 根本不具备触碰关键路径的能力。

## 问题：误删通常怎么发生

实际踩过坑的人会知道，Agent 误删很少是因为“它想删”，而是因为这些原因：

- **提示词注入**：网页内容或邮件里包含“请删除 /tmp/cache”之类的指令，模型照做。
- **路径拼接错误**：工具调用时 `base_dir + relative_path` 没有规范化，`../../` 逃逸出工作区。
- **命令参数误解**：模型以为 `rm -rf ./output` 是清理构建产物，实际当前目录是 `~`。
- **第三方插件权限过大**：MCP 插件注册了文件系统工具，但没声明只读。

所以 sandbox 要解决的不是“模型道德”，而是“执行环境约束”。

## 做法/步骤

OpenClaw 的 sandbox 配置通常放在 `openclaw.yaml` 中，以下是一个经过验证的最小配置示例：

```yaml
sandbox:
  enabled: true
  workspace: /home/user/openclaw-workspace
  readonly_mounts:
    - /home/user/Documents
    - /mnt/shared_data
  allow_commands:
    - ls
    - cat
    - grep
    - find
    - python3
  deny_commands:
    - rm
    - mv
    - dd
    - mkfs
  confirm_on_delete: true
  audit_log: /var/log/openclaw/audit.log
  follow_symlinks: false
```

启用后，Agent 只能看到 `workspace` 目录作为根，其他路径即使被模型提及，也会被 sandbox 拦截。`readonly_mounts` 让 Agent 可以读取参考资料，但无法写入。`deny_commands` 直接禁用高风险命令，`allow_commands` 则提供白名单，两者可以叠加，但建议只保留白名单。

如果业务确实需要删除操作，可以打开 `confirm_on_delete`，让每次删除都进入人工确认队列。对于重要目录，还可以在宿主机层用 zfs/btrfs 快照或 rsnapshot 做定时备份，OpenClaw 本身不负责回滚，但配合使用能形成完整恢复链路。

审计日志是最后一道防线。所有文件操作（包括被拒绝的）都会记录时间、Agent 会话 ID、工具名、参数和结果。出问题后可以从日志反向定位是哪个 prompt 或哪个插件触发的。

## 踩坑点

1. **路径穿越**：即使有 workspace，也要确认 OpenClaw 的 sandbox 做了路径规范化（realpath 或 jail）。测试时可以用 `ls ../../etc` 这类命令验证是否被拦。
2. **符号链接**：workspace 内的软链接可能指向外部文件。如果 `follow_symlinks: false`，Agent 操作链接本身是安全的；但如果某个工具默认跟随链接，就可能越界。建议统一禁用跟随，或定期扫描 workspace 内链接。
3. **相对路径与 `~` 展开**：模型可能会写 `~/.ssh` 或 `../config`。sandbox 应强制所有路径基于 workspace 解析，否则会踩坑。
4. **插件/MCP 权限**：OpenClaw 的插件体系是扩展点，但也是漏洞点。安装插件前要检查其权限声明，尤其是文件系统、网络、shell 类工具。不要给插件比主 Agent 更高的权限。
5. **dry-run 不是万能的**：有些命令支持 `--dry-run`，但很多自定义脚本不支持。不要依赖 dry-run 来替代 sandbox，它只是辅助验证。

## 可复用建议

- **最小权限原则**：只给 Agent 完成当前任务所需的最小路径和命令。任务结束可以回收或切换配置。
- **分层环境**：开发环境可以用宽松配置方便调试，生产环境必须收紧，甚至只挂载只读目录。
- **自动化误删测试**：写一个测试脚本，故意让 Agent 尝试删除 workspace 外的文件，断言 sandbox 拦截。这比人工验证可靠。
- **定期审计日志**：每周扫一次 audit_log，关注 `deny` 记录和异常删除操作，能提前发现配置漏洞或恶意 prompt。
- **把 Agent 当 CI 看待**：CI 也曾经有误删问题，后来靠隔离 runner、最小权限和审计解决。OpenClaw 同理，把它当成不受信任的执行单元。

## 总结

OpenClaw 的 sandbox 不是“防呆”，而是通过文件系统隔离、命令白名单、删除确认和审计日志，让 Agent 在受限环境中运行。它不能保证 100% 不误删，但能把误删半径从“整个宿主机”缩小到“一个可回滚的工作区”。真正安全的前提是：你像管理 CI 一样管理 Agent 的执行边界，而不是指望模型永远正确。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-27/2db27521f2db8647.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-27/a96417aeb689ad36.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-27/5b7d75e0196f3d25.png)

