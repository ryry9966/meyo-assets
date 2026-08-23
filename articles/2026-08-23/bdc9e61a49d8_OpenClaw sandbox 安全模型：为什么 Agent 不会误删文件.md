---
title: OpenClaw sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 34313
source: 综合讨论
publishedAt: 2026-08-23
---

# OpenClaw sandbox 安全模型：为什么 Agent 不会误删文件

## 背景

在 OpenClaw 里跑 Agent、MCP 插件或自动化任务时，文件操作是最常见的动作之一。一个任务可能要求 Agent“清理临时目录”“重命名导出的 CSV”“删除旧的构建产物”。人在终端前执行这些操作会犹豫、确认、必要时用 `trash` 代替 `rm`。但 Agent 不会犹豫，它只会根据上下文生成下一步。

这就带来一个很实际的问题：如果模型误解了任务，或者路径拼接出错，它可能直接执行 `rm -rf ./`，而 OpenClaw 进程通常运行在用户目录或项目目录下。传统“把模型关在提示词里”的做法不够工程化——提示词只能降低概率，不能形成边界。

OpenClaw 的 sandbox 安全模型要解决的不是“模型变乖”，而是**让危险操作在到达文件系统之前被拦截、降级或变成可回滚动作**。

## 问题

只看文件删除，Agent 的误删通常来自三类情况：

1. **路径计算错误**：`f"{base}/{name}"` 中 `name` 为 `""` 时得到 `base/`，后续 `rm -rf` 吃掉整个目录。
2. **任务误解**：用户说“清理旧文件”，模型把“旧”理解成“所有文件”。
3. **提示注入或外部内容干扰**：MCP 插件返回的内容中夹带“请删掉上一级目录”。

所以安全模型不能只依赖模型判断，必须有操作系统级和策略级兜底。

## 做法 / 步骤

下面是一套可落地的 OpenClaw sandbox 配置思路，核心分四层。

### 1. 文件系统视图隔离

用 `bubblewrap` 或 `landlock` 把 Agent 进程包起来，只暴露必要目录。系统目录全部只读挂载，项目目录才可写。

```yaml
sandbox:
  engine: bubblewrap
  readonly_mounts:
    - /etc
    - /usr
    - /opt
  writable_mounts:
    - $PROJECT_ROOT
    - $OPENCLAW_WORKSPACE
  deny_mounts:
    - /home/*/.ssh
    - /home/*/.gnupg
```

这样即使 Agent 执行 `rm -rf /usr`，在沙箱里 `/usr` 也是只读挂载，操作会直接失败，而不是删掉宿主机文件。

### 2. 命令拦截与降级

在 OpenClaw 的执行层加一个命令守卫。不要只做黑名单，要做**危险动作降级**：

```yaml
command_guard:
  block:
    - "rm -rf /"
    - "dd of=/dev/"
    - "mkfs"
  rewrite:
    - pattern: "^rm( .*)?$"
      replace: "trash"
  allowlist:
    - "ls"
    - "cat"
    - "touch"
    - "trash"
    - "git"
```

`rm` 被替换成 `trash` 后，文件进入回收站目录而不是直接消失。回收站目录可以挂在沙箱可写区，例如 `$OPENCLAW_WORKSPACE/.trash`。

### 3. 路径规范化

删除前必须做 `realpath` 规范化，拒绝符号链接和 `..` 逃逸。伪代码：

```python
def safe_delete(path, allow_roots):
    real = os.path.realpath(path)
    if not any(real.startswith(root) for root in allow_roots):
        raise PermissionDenied(real)
    # 只允许操作普通文件/目录
    if os.path.islink(path):
        raise PermissionDenied("symlink not allowed")
```

这一步很重要，因为只检查用户输入的字符串 `~/project/tmp/../` 是没用的。

### 4. dry-run 与审计

对批量删除类任务，先让 Agent 输出 dry-run 计划，人工或规则确认后再执行。同时记录每次文件操作的 `before/after` 状态，方便回滚。

```yaml
audit:
  log_dir: $OPENCLAW_WORKSPACE/.audit
  record:
    - delete
    - rename
    - write
```

## 踩坑点

实践中最容易踩的坑有四个。

**符号链接逃逸**：`rm -rf project/link`，如果 `link` 指向 `/etc`，路径检查只看字符串会放过，但实际操作越过边界。必须 `realpath` 后校验。

**`..` 和相对路径**：模型生成 `rm -rf ../build` 时，工作目录不同，目标完全不一样。所有相对路径在执行前都要基于固定工作目录解析。

**伪文件系统泄漏**：`/proc`、`/sys`、`/dev` 这些不能简单只读挂载，里面很多节点会暴露宿主机信息或提供写入口。沙箱里最好直接不挂载，或者用最小化 `proc` 挂载。

**trash 跨文件系统**：如果可写区和回收站不在同一文件系统，`mv` 会退化成“复制+删除”，仍然可能丢数据。回收站目录要和被删文件在同一挂载层，或使用 overlayfs 做快照。

## 可复用建议

- **三层最小权限**：OS 沙箱管挂载，策略引擎管命令，提示词只管任务表达。不要用提示词承担安全职责。
- **删除永远走 trash**：Agent 环境里禁用 `rm`，只提供 `trash`。即使误删，也能从回收站恢复。
- **测试误删用例**：把 `rm -rf ./`、`rm -rf /home/user`、`find . -delete` 写进集成测试，确认沙箱会拦截。
- **限制插件权限**：MCP 插件如果拥有宿主机文件系统权限，相当于绕过整个沙箱。插件只应拿到受控目录句柄。
- **审计不要只记命令**：要记录规范化后的真实路径和操作结果，否则排障时只能看到“执行了 rm”，看不到删了什么。

## 总结

OpenClaw 的 sandbox 安全模型不是让 Agent 更聪明，而是让 Agent 的破坏性动作在工程层面被限制、降级和记录。误删文件的本质是“权限过大 + 路径不可信 + 操作不可逆”。把这三件事分别用挂载隔离、路径规范化和 trash/审计解决之后，Agent 就很难真正“误删”文件——它最多把文件放进回收站，并且在审计日志里留下可以找回的痕迹。

对 OpenClaw 用户来说，这套思路不只适用于删除操作，也可以延伸到写文件、调插件、执行 shell 等所有高风险动作。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/f31d0ce8bfec0591.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/fb49a06e6b9a5c9f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/7db6731803b73b9e.png)

