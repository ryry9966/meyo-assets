---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 34957
source: 综合讨论
publishedAt: 2026-08-28
---

## 背景

在 OpenClaw 上跑 Agent、MCP 工具或插件时，文件系统操作是最容易出事故的一环。一个看似无害的清理脚本、一次路径拼接错误，或者插件内部调用了 `rm -rf`，都可能让项目目录瞬间消失。只靠提示词约束 Agent“不要乱删文件”并不可靠：模型可能理解偏差，插件也可能绕过对话上下文直接操作文件。

OpenClaw 的 sandbox 安全模型要解决的正是这个问题：让 Agent 拥有完成任务所需的最小文件访问能力，同时把误删、越权写入和路径逃逸的风险控制在可预期范围内。

## 问题

常见的文件事故有三类：

1. **误删项目文件**：Agent 执行清理、重构或临时文件删除时，把源文件、配置或 `.git` 目录删掉。
2. **路径穿越**：插件或 MCP 工具用相对路径或拼接路径访问工作区之外的敏感文件。
3. **越权覆盖**：写入时没有区分“允许修改的目录”和“只读目录”，导致系统配置或依赖被覆盖。

如果只依赖运行时提示，这三点都无法稳定兜底。沙箱的价值在于把限制下沉到文件系统层，而不是停留在模型层。

## 做法 / 步骤

一个可用的 OpenClaw sandbox 配置通常从三个层面入手：工作区白名单、只读挂载、删除拦截。

### 1. 限制工作区边界

在沙箱配置里显式声明允许读写的目录，而不是默认放开整个用户目录。典型配置如下：

```yaml
sandbox:
  enabled: true
  workspace: /home/user/project
  writable:
    - /home/user/project/src
    - /home/user/project/.openclaw
    - /tmp/openclaw-cache
  readonly:
    - /etc
    - /usr
    - /home/user/project/.git
```

这样 Agent 只能写 `workspace` 下的指定子目录。`/etc`、`/usr` 以及 `.git` 目录保持只读，即使执行了删除或覆盖操作也会被直接拒绝。

### 2. 删除操作重定向到回收站

沙箱可以拦截 `unlink`、`rmdir` 等删除类系统调用，把目标文件移动到指定的 `trash_dir`，而不是真正删除。配置里通常是：

```yaml
sandbox:
  trash_dir: /home/user/.openclaw/trash
  deny_delete:
    - /home/user/project/.git
    - /home/user/project/package-lock.json
```

这样即使 Agent 调用了 `rm -rf`，文件也只是被移动到一个隔离目录，不会物理消失。配合日志，可以清楚看到是哪次操作触发了删除。

### 3. 插件 / MCP 权限声明

插件式扩展需要单独声明文件系统权限。比如一个 MCP 工具只做日志读取，就不应该申请 `filesystem.write` 或 `filesystem.delete`。审计插件 manifest 时，建议只保留最小权限：

```json
{
  "name": "log-reader",
  "permissions": ["filesystem.read"]
}
```

如果插件尝试写入或删除，沙箱会拒绝并记录调用栈，便于排查是 Agent 的操作还是插件的行为。

## 踩坑点

实际使用中，有几个容易忽略的地方：

1. **白名单过宽**。把整个 `/home/user` 设成可写，等于没有沙箱。工作区必须细化到具体项目目录。
2. **符号链接逃逸**。即使限制了工作区，符号链接仍可能指向外部路径。需要在沙箱策略里禁止在工作区内创建指向外部的符号链接，或对解析后的真实路径做校验。
3. **`/tmp` 共享问题**。`/tmp` 通常是全局可写的。如果多个 Agent 共用同一个 `/tmp` 目录，可能发生误删或覆盖。建议给每个沙箱分配独立的临时目录。
4. **删除拦截不拦截覆盖写入**。`rm` 被拦住，但 `truncate`、`write` 仍然可以清空文件内容。沙箱策略需要同时覆盖写入权限，而不是只关注删除调用。
5. **回收站占用磁盘**。长期运行后回收站可能积累大量文件。需要定期清理，或者给 `trash_dir` 设置容量上限。

## 可复用建议

- **最小工作区原则**：每个任务只挂载必要的目录，任务结束后销毁沙箱。
- **只读优先**：不准备修改的目录一律设为 `readonly`。
- **开启回收站**：把删除重定向作为默认策略，而不是依赖提示词约束。
- **审计插件权限**：定期检查 MCP / 插件 manifest，移除多余的 `write`、`delete` 权限。
- **测试删除路径**：在沙箱内故意执行 `rm -rf`、路径穿越和覆盖写入，观察是否被成功拦截。

## 总结

OpenClaw 的 sandbox 安全模型不只是“阻止删除”，而是通过工作区白名单、只读挂载、删除重定向和插件权限声明，把文件操作限制在一个可控边界内。这样 Agent 即使判断失误，也不会直接造成不可逆的文件丢失。对工程化使用来说，沙箱配置应当和提示词约束配合使用：模型负责意图判断，沙箱负责底线兜底。

真正可靠的安全边界，不能只靠模型“自觉”，而要靠它想越界时，根本碰不到不该碰的文件。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/5156cc2bf0854240.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/b0339aa06e0c6016.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/352e335b431bfb94.png)

