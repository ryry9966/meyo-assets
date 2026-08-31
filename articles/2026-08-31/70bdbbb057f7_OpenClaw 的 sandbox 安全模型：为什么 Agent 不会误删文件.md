---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 35516
source: 综合讨论
publishedAt: 2026-08-31
---

## 背景

在 OpenClaw 上跑文件整理、批量重命名、日志清理或 MCP 工具时，最担心的不是 Agent 做不了，而是它“太敢做”。一个错误的目标路径、一段失控的 `rm -rf`、插件拿到过宽的工作目录，都可能让本地文件直接消失。

OpenClaw 的 sandbox 不是传统虚拟机或容器级隔离，更接近一个**可执行策略层 + 动作拦截层**。它的目标不是把 Agent 关进完全与外界无关的黑盒，而是让 Agent 的读写、删除、覆盖等动作经过明确边界和授权检查。

## 问题路径

常见误删主要来自三条路径：

1. **路径越界**：Agent 生成了 `../../` 或绝对路径，跳出工作目录。
2. **命令直通**：通过 shell 执行 `rm`、`move`、`shutil.rmtree` 等，没有经过权限判断。
3. **MCP/插件权限过大**：第三方 MCP server 拿到整个用户目录，错误工具调用直接破坏外部文件。

## 做法/步骤

在 OpenClaw 中，建议把 sandbox 配置成一套组合策略。

### 1. 建立独立 workspace

不要用 `~` 或系统根目录作为工作区。单独建目录：

```bash
mkdir -p /home/user/agent-workspace/projects
```

### 2. 限制可写路径

OpenClaw sandbox 支持把写入范围限制在 workspace 内。即使 Agent 尝试写外部路径，动作层也会拒绝：

```yaml
sandbox:
  root: /home/user/agent-workspace
  writable_paths:
    - /home/user/agent-workspace/projects
  allow_read_outside: false
```

### 3. 对删除类动作开启拦截

不要直接给永久删除权限。优先使用回收站策略，并要求确认：

```yaml
sandbox:
  delete_policy: trash
  confirm_required:
    - delete
    - move
    - overwrite
  default_allow: false
```

这意味着 `rm`、覆盖写、移动路径等动作会先进入待确认状态，Agent 必须给出目标、来源和原因，由策略层或人工确认。

### 4. 先 dry-run，后真实执行

批处理文件和清理任务先跑 dry-run。OpenClaw 的 action layer 可以只记录动作计划，不实际落盘：

```yaml
sandbox:
  dry_run: true
```

查看动作日志，确认没有越界路径或误匹配后再切换为 `dry_run: false`。

### 5. MCP/插件按最小权限注册

MCP tool 和插件不要挂载整个 home。给每个工具显式声明可用目录和动作：

```yaml
mcp_tools:
  - name: file_cleaner
    allowed_paths:
      - /home/user/agent-workspace/projects
    allowed_actions:
      - read
      - move_to_trash
    allow_delete: false
```

## 踩坑点

- **symlink 逃逸**：如果 workspace 里存在软链接指向外部目录，仅靠根目录限制可能被绕过。需要禁止跟随 workspace 外的 symlink，或对 symlink 单独授权。
- **dry-run 不是万能的**：dry-run 只能发现动作计划中的问题，无法覆盖真实执行时文件被占用、权限变化、目录已改变等情况。
- **trash 不代表安全**：如果某些外部工具绕过 OpenClaw 策略层直接操作文件，回收站策略不会生效。所有文件操作都应从 Agent 的动作通道走。
- **不要只靠 prompt 约束**：在系统提示词里写“不要删除文件”不能作为安全边界。必须让策略层成为默认拒绝的关卡。
- **日志和恢复**：开启动作审计日志。出现误操作后，先停止 Agent，根据日志确认动作链，从 trash 或 snapshot 恢复，而不是再让 Agent 尝试“修回去”。

## 可复用建议

1. 默认拒绝，显式允许。
2. workspace 独立、可写范围最小化。
3. 删除动作一律走 trash，不直接永久删。
4. 破坏性动作先 dry-run，再小批量执行。
5. MCP/插件按工具粒度授权，不允许继承整个用户目录。
6. 保留动作审计日志，做可回滚的 snapshot。

## 总结

OpenClaw 的 sandbox 之所以能让 Agent 不那么容易误删文件，核心不是“Agent 变聪明了”，而是文件操作被塞进了一个默认拒绝的通道。删除、覆盖、移动都必须经过显式策略、路径边界和确认机制。即便如此，误删只能被降低概率，不能被完全消灭。工程上的安全来自边界设计、最小权限和可恢复能力，而不是对模型判断力的信任。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/08a08dd43e99112c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/71e8175052c5be25.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/4f766bdf2417d20d.png)

