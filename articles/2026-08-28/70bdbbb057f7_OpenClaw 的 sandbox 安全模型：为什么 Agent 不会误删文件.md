---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 35070
source: 综合讨论
publishedAt: 2026-08-28
---

## 背景

在 OpenClaw 上跑文件整理、批量重命名、日志清理等自动化任务时，最容易让人犹豫的不是 Agent 能力不够，而是“它会不会把不该删的删了”。尤其是接入了 MCP 文件系统工具或本地插件后，误删风险会被进一步放大。

我自己在测试 OpenClaw 的 Agent 自动整理下载目录时，第一件事就是确认它的 sandbox 边界到底卡在哪一层。试下来发现，它的模型比想象中保守很多。

## 问题拆解

Agent 误删通常不是删除动作本身有问题，而是三个环节出了差错：

- **路径解析错误**：Agent 把相对路径或拼错的绝对路径解析到了用户主目录，而不是任务工作目录。
- **权限边界不清晰**：Agent 拿到了过于宽泛的文件系统权限，能写能删的范围比任务实际需要大得多。
- **删除操作缺乏回收机制**：直接物理删除，出错后无法恢复。

OpenClaw 的 sandbox 模型正是围绕这三点设计的。

## sandbox 安全模型

OpenClaw 的 sandbox 不是简单的 `chroot`，而是三层视图叠加：

1. **进程级隔离**：Agent 执行 exec / 文件操作时，默认运行在受限进程上下文，不能直接访问宿主机的 `/etc`、`~/.ssh` 等敏感路径。
2. **文件系统视图**：通过 `sandbox.root` 指定 Agent 可见的根目录，所有相对路径都解析到这个根下，外部文件不可见。
3. **操作审计与写保护**：写操作必须匹配 `sandbox.writable` 白名单；删除默认走回收站，而不是物理删除。

我实际配置如下（示意）：

```yaml
sandbox:
  root: /home/user/agent-workspace
  writable:
    - /home/user/agent-workspace/tmp
    - /home/user/agent-workspace/output
  delete_mode: trash
  follow_symlinks: false
  dry_run: true
```

这样配置后，即使 Agent 执行 `rm -rf /important`，其路径也会被解析到沙箱根下，外部文件不可见；如果不在白名单，写操作直接拒绝；删除进入 `~/.openclaw/trash` 而不是物理删除。

## 做法/步骤

- **第一步：为每个任务创建独立工作目录**，不要复用主目录，避免沙箱根被放大到整个 `/home/user`。
- **第二步：配置 `writable` 白名单**，只给任务需要的目录，其他目录默认只读或不可见。
- **第三步：开启 `dry_run`**，先跑计划，确认无越界操作后再执行正式任务。
- **第四步：删除操作全部走 `trash`**，保留 7 天，定期清理但同时保留误删恢复窗口。
- **第五步：定期查看审计日志**，定位异常文件访问或越界尝试。

## 踩坑点

1. **符号链接逃逸**：如果工作目录里有一个软链接指向 `/etc/passwd`，而 `follow_symlinks` 被误开，Agent 可能追出去。务必关闭该选项，或使用 `openat2` 的 `RESOLVE_BENEATH` 策略（新版 OpenClaw 默认启用）。
2. **插件绕过 sandbox**：有些 MCP 插件自带文件读写实现，不走 sandbox 的文件系统视图。需要单独审查插件权限，尽量用官方文件系统工具，避免第三方插件直接操作宿主机路径。
3. **不要用 root 跑 Agent**：容器或宿主机上如果用 root，即使有 sandbox 也可能被提权绕过。建议用非特权用户，并在容器内限制 capabilities。
4. **回收站被定期清空**：如果配置了自动清理脚本，别把 `trash` 目录也清掉，否则误删后无法恢复。
5. **只读挂载假象**：以为把某个目录挂成只读就安全，但文件删除依赖父目录的写权限。如果父目录可写，子目录即使只读也可能被整体删除，需要检查目录树权限。

## 可复用建议

- **目录分层**：`input`（只读）、`tmp`（可写）、`output`（可写）、`trash`（仅 Agent 可写）。这样 Agent 无法修改原始输入，只能处理副本。
- **任务前 dry-run，任务后 diff 文件树**：对比前后变化，确认删除/修改都符合预期。
- **用容器把 OpenClaw 实例和宿主机隔开**，即使 sandbox 失效也只影响容器内部。
- **关键文件用版本控制或快照**，不要依赖 sandbox 本身做备份。sandbox 是防线，不是备份策略。

## 总结

OpenClaw 的 sandbox 不保证“Agent 不会犯错”，而是通过缩小可见范围、白名单写入口、删除回收和审计，把误删的爆炸半径降到最小。工程上真正稳妥的方法是：**沙箱 + dry-run + 回收站 + 容器隔离**，四者叠加。不要因为有了 sandbox 就把整个 home 目录挂进去——那等于把防线拆了。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/ad25db5c83c2a01b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/a48364802c4f9420.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/22098f7de15f800b.png)

