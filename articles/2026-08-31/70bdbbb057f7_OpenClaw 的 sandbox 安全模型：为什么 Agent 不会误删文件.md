---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 35513
source: 综合讨论
publishedAt: 2026-08-31
---

## 背景

不少 OpenClaw 用户把文件读写、shell、MCP 工具接给 Agent 后，最担心的不是“能力不够”，而是误操作：把源码目录当临时目录删掉、覆盖配置、执行一条看起来安全的 `rm -rf`。OpenClaw 的应对方式不是让模型“更小心”，而是给工具调用增加一层可验证的边界。

## 问题不在模型一次都不出错

Agent 的工具调用来自上下文推理，路径可能由模型生成，也可能来自插件或 MCP 返回。问题在于：如果本地脚本以当前用户身份运行，shell 工具就能碰到任何路径。模型一旦把 `~/repo` 和 `~/repo/backup` 理解反了，传统脚本会直接执行。真正的风险不是 AI 会犯错，而是错误发生时系统没有拦截点。

## 做法/步骤

OpenClaw 的 sandbox 安全模型可以拆成三层：路径边界、操作策略、人工审批。

1. **固定可访问根目录**  
   文件工具只接受配置的 `WORKSPACE_ROOT` 或 `ALLOWED_ROOTS` 下的路径。工具执行前先做路径规范化，比较前缀；遇到 `..`、绝对路径逃逸、符号链接指向外部路径时直接拒绝。

2. **删除动作改写**  
   危险的 `delete` 不直接 `unlink`。OpenClaw 可以将删除动作改写为移动到带时间戳的 `.trash` 目录，例如 `~/.openclaw/trash/2026-...`。只有显式开启“硬删除”且通过审批时才执行真正的 `rm`。

3. **按操作分级审批**  
   读、写、覆盖、删除四种操作可以配置不同策略。常见配置是：读自动放行，写受限，覆盖和删除需要人工确认，批量删除需要额外确认。

4. **MCP 工具白名单**  
   不是所有 MCP server 都值得信任。OpenClaw 侧只暴露白名单工具，并限制参数范围。尤其不要默认把 `shell.run`、`fs.delete`、`fs.move` 全部开放给 Agent。

示意配置如下，实际字段名以当前版本为准：

```yaml
sandbox:
  roots: ["~/openclaw-data"]
  deny_ops: ["rm", "rmdir", "unlink"]
  trash_dir: "~/.openclaw/trash"
  require_approval: ["delete", "overwrite"]
```

## 踩坑点

- **根目录给太大**：把 `~` 或 `/` 加进 roots，等于没设边界。
- **符号链接逃逸**：`/data/link` 可能指向 `/etc`。要先 `resolve` 真实路径，再做前缀比较。
- **MCP server 不一定遵守同一 sandbox**：外部 MCP 进程可能用自己的用户和路径权限，应放进容器或限制其可访问路径。
- **shell 别名绕过**：就算 `delete` 被拦截，`bash -c 'rm -rf ...'` 仍能绕过文件工具。需要禁止 shell 或让 shell 在更底层隔离环境中运行。
- **审批疲劳**：所有删除都弹确认，用户会逐渐无脑点同意。应只对高风险目录启用人工审批，低风险目录用 trash 兜底。

## 可复用建议

- Agent 的数据目录不要直接放在代码仓库里，使用独立的 `~/openclaw-agent-data`。
- 重要目录保留版本控制或定时快照，sandbox 不是备份替代品。
- 放一个 canary 文件，如 `DO_NOT_DELETE`，定期测试工具是否真的拒绝删除。
- 每周抽看一次审计日志，关注“拒绝的操作”和“已进入 trash 的文件”。
- sandbox 是权限边界和调度策略，不是内核隔离。要执行不可信命令，仍需容器、虚拟机或 landlock/seccomp 等机制。

## 总结

OpenClaw 的 sandbox 模型不是承诺“AI 永不犯错”，而是把文件操作变成受限请求：路径不对就拒绝，删除先入 trash，高风险操作需要人工确认。因此“不会误删文件”更准确的说法是：未经允许的路径删不掉，已授权的删除可恢复、可审计。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/4219c4780b3d1131.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/d494a3fbd9c1ef18.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/7b352b9e5f6cb2be.png)

