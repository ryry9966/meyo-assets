---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 36855
source: 综合讨论
publishedAt: 2026-09-10
---

## 背景

让 Agent 直接操作文件系统，是自动化收益最大、也最容易出事的场景。OpenClaw 的 Agent 默认带文件读写和 shell 类工具，社区里被问得最多的问题之一就是："它会不会哪天把我整个目录删了？"这篇文章拆一下 OpenClaw 的 sandbox 模型。先说结论：**"不误删"不靠模型自觉，靠的是一组默认生效的机制。**

## 问题

模型侧的风险是真实的：路径幻觉（把 `~/projects` 写成 `~/`）、glob 过宽（清理 `*.log` 结果匹配到源码）、插件工具权限过宽、子进程逃逸。这些不是假设，群里就有人贴过 Agent 在错误工作目录执行 clean 的案例。单靠 prompt 约束不可靠，所以 OpenClaw 把安全放在工具层和系统层，而不是指望模型"懂事"。

## 做法：五层闸门

1. **工作区 jail**：每个会话绑定一个 workspace root，所有文件工具的路径先做 canonicalize（解析 symlink、展开 `..`），越出 root 的读写直接拒绝。这是最底层兜底。
2. **操作分级**：读、写、删是三种权限。删除类操作（unlink、递归删除、截断覆盖）默认不授予权限，需要显式开启。
3. **路径策略**：allowlist 只放行 workspace 内路径；denylist 硬挡 `~/.ssh`、`~/.config`、`/etc` 等高危位置，且不可被会话内指令覆盖。
4. **软删除 + 审计**：sandbox 内的删除不直接 unlink，先移入 `.openclaw/trash/<session>/`，同时写 audit log（时间、原路径、调用工具）。会话内可一键回滚。
5. **危险操作 dry-run**：批量删除或覆盖前，工具层强制先产出 affected-files 列表，确认后才执行。

MCP/插件侧，每个工具的 manifest 必须声明 scope（`fs:read` / `fs:write` / `fs:delete`），sandbox 在调用时校验，声明与实际行为不符会被拒绝并告警。

典型配置：

```yaml
sandbox:
  root: ./workspace
  follow_symlinks: false
  policy:
    fs:delete: confirm   # confirm | deny | allow
  deny:
    - ~/.ssh/**
    - ~/.config/**
  trash: .openclaw/trash
```

## 踩坑点

- **symlink 逃逸**：workspace 里一个软链指向 `/home`，旧版本 `follow_symlinks` 默认开启。请显式设为 `false`。
- **子进程绕过**：Agent spawn `bash -c "rm ..."` 走的是系统层 jail 拦截，但如果你同时给了 execute 权限又没限 root，等于白设。生产会话建议不给 execute，用受控的 run 工具。
- **插件私带 fs 实现**：有的插件直接调裸文件 API 不走 gate，manifest 校验拦不住行为不符，只能靠装前审计——看它用的是官方 toolkit 还是裸 API。
- **glob 展开位置**：展开发生在 sandbox 内部不会越界；旧版本交给外部 shell 展开会"先炸再拦"，升级可解。
- **临时目录误报**：tmpdir 配在 workspace 外，批量生成任务会被拦，属策略误报，把 tmpdir 挪进 root 或加入 allowlist。

## 可复用建议

- 默认最小权限，destructive 永远显式开；
- 上线前跑一组红队用例：让 Agent 尝试写 root 外、删 denylist 文件、经 symlink 越界，预期全部失败且 audit 有记录；
- 定期 review 插件 manifest 的 scope 声明；
- trash 和 audit log 别关，出事后它们是唯一的取证手段。

## 总结

"Agent 不会误删文件"，不是因为模型足够聪明，而是最短路径上每一步都有闸门：路径规范化挡越界，权限分级挡误操作，软删除和 dry-run 兜住不可逆，manifest 校验约束生态。分层防御的意义在于：任何一层失效，都不会直接变成事故。建议大家回头检查自己的 sandbox 配置，尤其是 `follow_symlinks` 和 `fs:delete` 这两项。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/16d3393bde42261a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/17ea040ae7f5afb6.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/25ff62841e866dfc.png)

