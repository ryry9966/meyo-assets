---
title: OpenClaw 沙箱安全模型：为什么 Agent 不会误删文件
feedId: 34985
source: 综合讨论
publishedAt: 2026-08-28
---

## 背景

不少刚把 Agent 接入本地目录的用户，最担心的是模型一句“我来清理临时文件”，就把项目删了。这个恐惧不是没道理：Agent 的规划来自模型，模型对路径的“记忆”并不可靠，而插件或 MCP 工具如果直接暴露 shell，等于把删除能力交给了概率系统。

OpenClaw 的沙箱安全模型做了几层隔离，让删除从“不可逆事故”变成“可拦截、可回滚的操作”。它要解决的不是让 Agent 变聪明，而是在工具层切断风险来源。

## 问题：误删通常来自三条路径

一是**路径幻觉**：模型生成了 `/`、`~` 或错误相对路径。二是**提示注入**：网页、邮件、聊天上下文里的指令诱导 Agent 执行删除。三是**插件越权**：一个本来只读数据的 MCP 插件请求了 write/delete 权限。

如果所有文件操作都直接走真实文件系统，任何一层风险都可能变成事故。沙箱的核心是让“删除”不再是一个默认能力。

## 做法：默认拒绝 + 显式例外 + 软删除

以 OpenClaw 的沙箱配置为例，文件系统规则默认是 `deny`，只有显式声明权限才会放行。注意：**删除权限独立于写权限**，不是拥有 `write` 就自动能删。

```yaml
filesystem:
  default_policy: deny
  rules:
    - path: "workspace/**"
      permissions: [read, write]
    - path: "workspace/tmp/**"
      permissions: [read, write, delete]
    - path: "data/**"
      permissions: [read]
  delete:
    mode: trash          # 删除前先移入 .openclaw/trash
    max_batch: 20
  symlinks: false
  commands:
    allow: ["ls", "cat", "grep", "find", "wc"]
    shell: false
```

这段配置的含义是：

- Agent 能读写 `workspace`，但只有 `workspace/tmp` 下能删。
- `data` 目录只读，即使模型想删 `data/important.csv`，也会被工具层拒绝。
- 删除不会直接 `unlink`，而是移入 `.openclaw/trash`，可恢复。
- 禁止跟随符号链接，防止 workspace 下放软链指向 `/etc` 逃逸。
- shell 默认关闭，只允许只读命令；任何删除操作必须走受控的 fs 工具。

验证时可以用 dry-run 模式观察路径是否会被放行：

```bash
openclaw sandbox dry-run "remove workspace/tmp/delete.txt"
# 输出：allowed: delete -> trash workspace/tmp/delete.txt

openclaw sandbox dry-run "remove data/important.csv"
# 输出：denied: delete permission not granted
```

## 踩坑点

1. **把路径配太宽**。给 `data/**` 加了 `delete`，或者把 workspace 设为 `/`，等于自行关闭沙箱。
2. **开启 `symlinks: true`**。规则检查的是符号链接路径而不是真实路径，容易逃逸。务必保持 `symlinks: false`。
3. **trash 模式的语义差**。Agent 以为文件“已删除”，实际还在 `.openclaw/trash`。后续逻辑可能重复处理，或把清理误判为成功。需要在系统提示里明确：删除是软删除。
4. **第三方插件绕过内置工具**。一些 MCP 插件会直接请求 shell。插件权限必须单独限制，不能因为“听起来可信”就放开 delete。

## 可复用建议

- **最小权限**：读、写、删三种权限分开授予，批量删除加 `max_batch` 阈值。
- **非 root 运行**：Agent 服务账号不要用 root，沙箱之外再加一层容器或系统权限限制。
- **金丝雀文件**：在关键目录放一两个已知文件，定期检查是否被修改或删除，作为逃逸告警。
- **审计日志**：记录所有 delete/rename 事件，并在 CI 中用 dry-run 重放典型任务。
- **提示词兜底**：明确告诉 Agent 不要删除 workspace 根目录文件，除非用户显式要求。但安全模型不能只依赖提示词。

## 总结

OpenClaw 的沙箱安全模型并不是“Agent 永远不会删错”，而是通过默认拒绝、删除权限独立、trash 软删除、禁止符号链接和 dry-run 验证，让大多数误删在发生前被挡下，发生中可恢复。

把删除权限从“默认能力”改成“显式例外”，是 Agent 落地文件系统时最值得抄的一步。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/c33a16d59ef98bcf.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/6cc06ebfc46faeb5.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/9b711e7ac704b3ac.png)

