---
title: OpenClaw 沙箱安全模型拆解：Agent 为什么不会一刀切删库
feedId: 34080
source: 综合讨论
publishedAt: 2026-08-22
---

## 背景

在 OpenClaw 里接入 Agent 做自动化时，最让人不安的不是它写得慢，而是它“手太快”。一旦 Agent 具备文件操作能力，误删风险的来源通常不是模型“想删库”，而是三类工程问题：

- 路径解析错误：`../`、软链接、绝对路径拼错；
- 命令拼接问题：任务描述里出现 `rm -rf /tmp/xxx`，但 `xxx` 被意外解析为空；
- 上下文误导：Agent 被长对话带偏，把清理临时文件理解成清理整个工作目录。

只靠 prompt 约束不可靠，因为 prompt 是概率行为。OpenClaw 的设计思路是在执行层做约束，而不是在模型层做劝说。

## 问题：为什么不能直接给文件系统权限

如果让 Agent 直接以宿主机用户运行 shell，等于把 `.ssh`、`.gitconfig`、系统配置全部暴露在一条删除命令之下。常见的危险场景包括：

```bash
rm -rf /tmp / var/log
find / -name "*.log" -delete
python -c "import os; os.remove('/etc/passwd')"
```

这些命令不一定来自恶意，更多来自任务理解偏差或变量拼接错误。因此，安全边界必须下移到执行器。

## OpenClaw 的做法：默认拒绝 + 执行层代理

OpenClaw 的 sandbox 不是“智能判断能不能删”，而是把文件操作包在一层代理里。严格模式下，默认行为如下：

### 1. 白名单目录，而不是黑名单

策略配置大致如下：

```yaml
sandbox:
  mode: strict
  writable:
    - /opt/openclaw/workspace
    - /tmp/openclaw
  readonly:
    - /usr
    - /etc
  deny:
    - /home
    - /root
    - /
  trash: /opt/openclaw/.trash
```

Agent 只对 `workspace` 和 `tmp/openclaw` 有写权限。`/`、`/home`、`/root` 直接被 deny。不要因为“方便”就把整个 `$HOME` 暴露进去。

### 2. 删除动作进回收站，而不是直接 unlink

启用 `trash` 后，`rm`、`unlink` 会被代理层拦截，文件先移动到 `.trash/<timestamp>-<hash>`。这个机制给误删留了恢复窗口。

### 3. 限制通用 shell 和解释器

只禁用 `rm` 不够，Agent 可以绕过：

```bash
python -c "import os; os.remove('/tmp/x')"
perl -e 'unlink("/tmp/x")'
find /tmp -type f -delete
```

所以在严格模式下，应尽量避免直接给 `/bin/bash -c`。优先开放声明式的文件工具，删除权限必须单独声明。

### 4. dry-run 与审计

高破坏性操作可以先返回计划，等待人工确认。同时记录审计日志，字段至少包含：

- `agent_id`
- 原始命令
- 真实目标路径
- 是否被拦截
- 处理方式：deny / trash / allow

## 踩坑点

### 1. 软链接逃逸

即使限制了目录，如果 workspace 里存在一个软链接指向 `/etc`，Agent 执行删除仍可能影响宿主机。需要开启 `no_follow_symlink` 或在文件系统层限制 `O_NOFOLLOW`。

### 2. 相对路径穿透

`rm -rf ../` 从 workspace 向上逃逸是常见问题。代理层必须对真实路径做规范化，并校验是否仍在允许前缀内。

### 3. MCP 插件绕过 sandbox

很多插件自己走宿主机文件系统，不经过 OpenClaw 的文件代理。引入 MCP 插件前，要确认插件是否声明了文件读写权限，以及是否遵守 sandbox 边界。

### 4. 回收站被二次删除

回收站目录如果对 Agent 可见且可写，可能被一条 `rm -rf /opt/openclaw/.trash/*` 清空。回收站只应让宿主管理员可写。

### 5. 权限过宽

把 `/home` 或 `/` 设置为可写，基本等于没有 sandbox。遇到过一次“为了读日志方便把 `/var/log` 设成可写”，结果 Agent 清理日志时把整个目录权限改坏。

## 可复用建议

- **最小可写区**：新建专用 workspace，权限 `700`，不挂载 home；
- **只读正文，可写受限**：系统目录尽量只读，必要时用拷贝到 workspace 的方式处理；
- **禁止通用 shell**：优先给 Agent 提供受控工具，而不是自由 shell；
- **强制确认**：destructive 操作要求 approve，或至少 dry-run；
- **回归测试**：定期用危险用例集验证，如 `rm -rf /`、`find / -delete`、软链接删除等；
- **审计可追溯**：所有删除操作留痕，方便定位责任和恢复路径。

## 总结

OpenClaw 的 sandbox 安全模型不是让 Agent 更聪明，而是让它即使判断失误，也没有能力直接破坏宿主机。安全来自分层：默认拒绝、目录白名单、路径校验、系统调用限制、回收站和审计。配置得当，大多数误删都能被拦下；配置宽松，仍然可能翻车。真正的安全不是一次设置，而是持续约束和回归验证。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/c0be2981e3e99e85.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/aaef8df450665f14.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/d868e02182b0fa51.png)

