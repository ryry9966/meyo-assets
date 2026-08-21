---
title: OpenClaw sandbox 安全模型拆解：为什么 Agent 不会误删文件
feedId: 34099
source: 综合讨论
publishedAt: 2026-08-22
---

## 背景

在 OpenClaw 里跑 Agent 时，一个常见担忧是：Agent 为了完成任务可能执行 shell、读写文件，一旦模型输出错误指令，例如把 `rm -rf /tmp/*` 写成 `rm -rf / tmp/*`，或路径拼接出错，是否会把宿主环境删干净？实际部署中，我遇到过一个案例：Agent 想清理临时目录，结果因为变量为空，执行了 `rm -rf ${WORK_DIR}/*`，在 shell 展开后变成 `rm -rf /*`。好在沙箱拦住了。这篇文章拆解 OpenClaw 的 sandbox 安全模型，说明它是如何把“误删文件”的爆炸半径控制住。

## 问题：Agent 文件操作的风险点

Agent 与普通脚本不同，它的动作来自模型推理，不可完全预测。风险集中在三类：

1. **路径边界不清**：目标路径没有强制限制在工作目录内，可能出现 `../`、绝对路径、符号链接逃逸。
2. **危险命令直通**：`rm -rf`、`find -delete`、`mv` 覆盖、`dd` 写盘等命令直接落到宿主环境。
3. **无回滚能力**：文件一旦删除或覆盖，没有回收站或快照，排查和恢复成本很高。

如果只靠 prompt 约束“不要删文件”，工程上不可靠。OpenClaw 的做法是在执行层加一道 sandbox。

## 做法：三道防线

### 1. 限定根路径：所有文件操作先归一化

在 OpenClaw 的 sandbox 配置中，通常会设置一个 `workspace_root`，例如 `/var/openclaw/sandbox/<session_id>`。Agent 的所有文件工具（read/write/edit/delete）都会经过统一的路径解析函数：

```python
def safe_join(root: str, user_path: str) -> str:
    base = os.path.realpath(root)
    target = os.path.realpath(os.path.join(base, user_path))
    if not target.startswith(base + os.sep):
        raise PermissionError("path escape detected")
    return target
```

这里的关键不是简单拼接，而是 `realpath` 后再判断前缀。这样 `../`、符号链接、绝对路径都会被拦在根目录外。OpenClaw 对 shell 命令也类似：在执行前把 `cwd` 限制在 workspace 内，并对命令中的文件参数做同样的归一化校验。

实际配置参考：

```yaml
sandbox:
  root: /var/openclaw/sandbox/{{session_id}}
  readonly_mounts:
    - /etc
    - /usr
    - /opt/openclaw/runtime
  writable_mounts:
    - /var/openclaw/sandbox/{{session_id}}/workspace
    - /tmp/openclaw/{{session_id}}
  deny_commands:
    - "rm -rf /"
    - "find / -delete"
    - "mkfs"
    - "dd if=/dev/zero of=/dev/"
```

### 2. 写隔离：系统目录只读，工作目录可写

OpenClaw 的 sandbox 通常不是完整虚拟机，而是基于 namespace/chroot/overlay 的轻量隔离。宿主系统目录（如 `/etc`、`/usr`、`/bin`）以只读方式挂载，Agent 只对 workspace 和临时目录有写权限。这样即使命令中出现了 `/etc/passwd` 或 `/usr/local/bin`，写操作也会被文件系统层直接拒绝，而不是依赖命令过滤。

对于需要安装依赖的场景，可以给一个可写的 overlay 层，但该层在会话结束后可以丢弃。这样 Agent 即使执行 `pip install` 或 `apt-get`，也不会污染宿主环境。

### 3. 危险命令拦截 + 回收站

OpenClaw 在 shell 执行前会做一层命令审计。它不只用正则匹配 `rm -rf`，而是解析命令结构。例如：

```python
deny_patterns = [
    ("rm", ["-rf", "-fr", "--recursive", "--force"]),
    ("find", ["-delete"]),
    ("mv", ["/"]),
]
```

对于删除操作，OpenClaw 会优先把目标文件 move 到 `.trash/` 而不是直接 unlink。这样即使 Agent 删错，也能从回收站恢复。回收站可以按 session 隔离，定期清理。

## 踩坑点

在实际配置中，有几个容易忽略的地方：

1. **符号链接绕过**：如果只做字符串前缀判断，`/var/openclaw/sandbox/session1/link -> /etc` 会被认为安全。必须先 `realpath` 再判断。
2. **命令中的通配符**：`rm -rf /tmp/*` 可能因为变量为空变成 `rm -rf /*`。在 shell 展开前，OpenClaw 会检查变量是否为空，并对危险通配符告警。
3. **插件直接调用 `os.remove`**：如果插件运行在 Agent 进程内，没有经过 sandbox 的文件工具，就会绕过路径校验。因此插件必须声明权限，并在独立进程/namespace 中运行。
4. **overlay 层无限增长**：只读挂载 + overlay 会让可写层越积越多，需要定期清理或限制大小。
5. **回收站占用空间**：`.trash` 目录如果不清理，可能比 workspace 还大。建议按时间或大小做清理策略。

## 可复用建议

- **默认拒绝**：文件写操作默认只允许在 workspace 内，其他路径需要显式声明。
- **路径校验统一入口**：不要在每个工具里手写路径判断，抽成一个 `safe_join` 或 `resolve_in_root` 函数。
- **命令白名单优于黑名单**：对于常用操作，允许 `ls`、`cat`、`mkdir`、`cp` 等，其余默认走审批或拒绝。
- **开启 dry-run**：对删除、覆盖类操作，先输出将执行的命令和目标，让用户确认。
- **审计日志**：记录每次文件操作的原始命令、归一化后路径、结果，便于排障。
- **定期清理**：给回收站和 overlay 设置 TTL 或容量上限。

## 总结

OpenClaw 的 sandbox 安全模型不是“绝对防删”，而是通过根路径限制、只读挂载、命令审计和回收站四层机制，把 Agent 的误操作限制在可恢复的范围内。真正要避免误删文件，不能只靠模型自觉，也不能只靠一层正则，需要路径归一化、文件系统隔离和回滚机制配合。工程上，这套模型可以低成本复用：先划边界，再做隔离，最后留退路。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/dc5079ad9af0361b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/4b45048bd8b01679.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/cca1d7c536f4aa2d.png)

