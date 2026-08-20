---
title: OpenClaw sandbox 安全模型拆解：为什么 Agent 不会误删文件
feedId: 33908
source: 综合讨论
publishedAt: 2026-08-20
---

## 背景

给 Agent 接上文件系统后，最怕的不是它不会写代码，而是某次工具调用里多了解释不清的路径，然后一条 `rm -rf` 落在你的 `~/Documents` 上。很多 Agent 框架习惯用 prompt 约束模型“不要删除用户文件”，但 prompt 是概率行为，工程上不能当作安全边界。

OpenClaw 的 sandbox 走的是另一条路：不在模型意图上较劲，而是在执行层限制文件操作。它的目标不是让 Agent 变聪明，而是让它即使想删，也删不到不该删的东西。

## 问题

误删文件的来源通常不是模型“故意作恶”，而是几类工程问题叠加：

- 路径拼接错误，例如 `~/` 没有展开、相对路径指向了错误目录。
- MCP filesystem server 权限过大，Agent 可以访问整个 home 目录。
- 插件或自定义 MCP 绕过 sandbox，直接以用户权限执行删除。
- 删除操作直接落到真实文件系统，没有回收站缓冲。

所以“不会误删”不能靠信心，要看 sandbox 默认策略是否把这几个口子都堵上。

## 做法 / 步骤

OpenClaw 的 sandbox 默认模型可以拆成四层：工作目录隔离、删除重定向、只读挂载、MCP 权限收敛。

一个常见的配置示例如下，字段名以你的 OpenClaw 版本为准：

```yaml
sandbox:
  enabled: true
  fs:
    root: /home/user/openclaw-workspace
    readOnlyPaths:
      - /home/user/Documents
      - /etc
    deletePolicy: trash
    trashDir: /home/user/.openclaw/trash
mcp:
  filesystem:
    allowedDirectories:
      - /home/user/openclaw-workspace
    deniedOperations:
      - rm
      - rmdir
      - unlink
```

验证时可以用两个测试文件：

1. 在 workspace 内建一个 `test.txt`，让 Agent 删除它。
2. 在 workspace 外建一个 `outside.txt`，让 Agent 删除它。

预期结果是：workspace 内的文件被移动到 `trashDir`，而不是直接原地消失；workspace 外的文件返回 `permission denied` 或工具调用失败。这样就能确认删除动作被 sandbox 接管了。

检查日志时重点关注 `rm`、`unlink`、`rmdir`、`move to trash` 这几类事件。OpenClaw 的审计日志里通常会记录实际落盘路径，这比只看模型回复可靠得多。

## 踩坑点

实际配置中容易踩的坑有几个：

**软链接逃逸**  
如果 workspace 内有一个软链接指向外部目录，而 sandbox 只做了路径字符串判断，Agent 可能通过软链接读写外部文件。需要开启 realpath 解析，或者直接禁止在 workspace 内创建符号链接。

**`~` 展开不一致**  
配置文件里的 `~` 不一定被展开成 `/home/user`。如果 sandbox 和 MCP server 的 workdir 不同，`~` 可能指向不同位置。建议所有路径都写绝对路径，避免环境差异。

**MCP 独立进程权限**  
filesystem MCP server 如果在 sandbox 外启动，它自身拥有用户级权限。即使 OpenClaw 限制了工具调用，MCP server 本身仍然可能被其他入口调用。需要限制 MCP server 的运行用户，或在系统层面对目录做权限隔离。

**trash 不是备份**  
`deletePolicy: trash` 只是把删除变成移动操作，但如果 trash 目录本身可以被 Agent 写入或清空，保护就会失效。建议把 trash 目录放在 Agent 不可写的位置，或定期同步到冷存储。

## 可复用建议

- 默认只给 Agent 一个专门 workspace，不要直接挂载整个 home。
- 删除策略优先用 `trash` 或 `deny`，不要一开始就允许真实删除。
- 对重要目录做只读挂载，例如 `Documents`、`Code`、配置文件目录。
- MCP filesystem 同时限制 `allowedDirectories` 和 `deniedOperations`，不要只做一层。
- 开启审计日志，定期 grep `rm`、`unlink`，发现异常调用要及时收敛权限。
- 把删除行为测试放进 CI 或初始化脚本，每次升级 OpenClaw 后跑一遍。

## 总结

OpenClaw 的 sandbox 并不是靠一个开关就保证安全，而是把 Agent 的文件操作限制在“只能碰工作区、删除进回收站、外部路径只读”这几条边界里。模型仍然可能发起删除，但执行层已经替用户挡了一道。

如果你准备在真实数据目录上使用，建议先在临时目录完整验证路径、软链接、MCP 权限和 trash 恢复流程。把 sandbox 当成工程问题管理，比把安全寄托在模型自觉上，可靠得多。

---

