---
title: OpenClaw sandbox 安全模型：Agent 为什么不会误删文件
feedId: 35161
source: 综合讨论
publishedAt: 2026-08-29
---

## 背景

在 OpenClaw 上跑文件整理、批量重命名、代码生成或 MCP 插件任务时，最常被问到的问题是：Agent 会不会在某个瞬间执行 `rm -rf`，把整个项目删光？

答案是：默认不会。但这个“不会”不是因为它足够聪明，而是因为 OpenClaw 的 sandbox 把危险操作挡在边界外。对 Agent 来说，文件系统不是宿主机文件系统，而是一块受控、可回滚、可审计的沙箱视图。

## 问题

Agent 的工具调用本质上是由模型决定执行文件或 shell 操作。模型可能被上下文误导，可能生成错误路径，也可能受到 prompt injection。因此安全不能依赖模型自觉，必须依赖运行时约束。

但约束也不能过严。如果完全禁止写操作，Agent 的自动化价值会大幅下降。OpenClaw 的做法是把“能访问什么”和“怎么访问”拆开，形成多层边界，而不是一个简单权限开关。

## 做法 / 步骤

### 1. 工作区隔离

每个 Agent 会话启动时，会创建独立 workspace。Agent 看到的 `/workspace` 不是真实的 `~/myproject`，而是映射到沙箱内的可写层。宿主文件系统默认只读或不可见。

即便 Agent 执行 `rm -rf /workspace/*`，删除的也只是沙箱层里的文件，不会直接影响宿主机原目录。

### 2. 路径重写与权限声明

工具侧可以声明读写范围，例如：

```yaml
sandbox:
  fs:
    read: ["workspace/**", "config/**"]
    write: ["workspace/**"]
```

所有文件路径在进入宿主机前，会先经过 sandbox resolver 重写和校验。越界路径直接返回 `EPERM`。像 `/etc/passwd`、`~/.ssh` 这类路径，通常根本不在可解析范围内。

### 3. 删除保护

递归删除和覆盖操作默认需要显式允许。即使允许，文件也会先移动到沙箱 trash 或备份目录，任务结束前可恢复。还可以开启 dry-run，先列出会被影响的文件，再决定是否真正执行。

### 4. 进程与网络约束

文件沙箱通常叠加 seccomp、landlock 等机制，禁止 Agent 进程绕过工具层直接触碰敏感路径，也禁止 `mount`、`umount` 等提权操作。

### 5. 提交 / 回滚

任务完成后，Agent 对 workspace 的修改不会自动写回宿主机。可以选择 commit 合并变更，或 discard 丢弃沙箱层。类似 Git worktree，出错时能整体回滚。

## 踩坑点

### 不要用 root 运行 Agent

root 在部分沙箱实现中可能绕过文件权限位，也能执行 mount 等操作。OpenClaw 的 Agent 进程应使用低权限用户运行，否则沙箱边界可能被穿透。

### 注意符号链接逃逸

如果沙箱内允许创建 symlink 指向外部路径，后续写入可能逃逸到宿主机。OpenClaw 的 resolver 会拒绝跳出 workspace 的 symlink，但在 macOS 和 Windows 上沙箱语义不同，需要单独验证。

### 不要直接挂载敏感目录为可写

不要把宿主机 `~/.ssh`、`/data` 或整个 home 目录直接挂载为可写。应只读挂载，或先把数据复制到 workspace 再让 Agent 处理。

### 避免在宿主 shell 拼路径清理

清理沙箱时，不要写类似 `rm -rf $TMP_DIR/` 的宿主机命令。如果变量为空，可能造成灾难。应使用框架提供的 cleanup 命令。

### 删除保护不是万能的

如果 `allowDelete=true` 且 write 范围过大，Agent 仍可能删掉整个 workspace。删除保护只在配合最小权限时才有意义。

## 可复用建议

- **默认最小权限**：所有任务先只读，写操作单独声明目录。
- **开启 trash / dry-run**：批量删除或移动前，先 dry-run 查看影响列表。
- **记录审计日志**：记录工具调用的原始路径和重写后路径，便于排障。
- **按任务分配 workspace**：不要在生产目录上直接跑 Agent。
- **网关层拦截高风险指令**：`rm -rf /`、`sudo`、`mount`、`chmod 777` 等直接拒绝或转人工确认。

## 总结

OpenClaw 的 sandbox 安全模型不是“禁掉删除”，而是让删除发生在可回滚、可审计、受限的边界内。一次真正的误删，通常需要越界访问、删除保护、回滚机制三层同时失效。

把权限最小化、路径重写和回收站策略配好，Agent 才能在自动化任务里既干活又不闯祸。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/0ff60f8f6717605b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/50440c3c9cdece31.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/4bd936de215daf52.png)

