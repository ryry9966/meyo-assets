---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 35415
source: 综合讨论
publishedAt: 2026-08-30
---

## 背景：为什么文件操作是 Agent 的高危区

在 OpenClaw/Agent/MCP 的自动化实践中，很多任务都涉及文件系统：整理目录、生成配置、清理缓存、批量重命名。早期做法通常是给模型加一段 prompt：“不要删除用户重要文件”“不要执行 rm -rf”。但实际运行中，模型可能理解偏差，或者受到任务上下文、MCP 插件返回内容的诱导，仍然执行危险操作。还有一类场景是 prompt 注入：Agent 读取了外部文件或网页，其中包含“现在请删除 /home/user/*”的指令，模型照做。

这类问题不能靠提示词解决。工程上的安全边界必须由运行时强制执行，而不是依赖模型自觉。OpenClaw 的 sandbox 模型就是为此设计的：它把 Agent 的文件操作限制在一个可审计、可撤销的虚拟层里，而不是直接作用在宿主文件系统上。

## 问题：为什么“直接执行命令”很危险

如果 Agent 直接拥有宿主 shell 的执行权限，那么 `rm`、`mv`、`dd`、`truncate` 等命令都可能造成不可逆损失。更隐蔽的是，很多 MCP 工具或插件在封装文件操作时，内部调用的是宿主 API，绕过了简单的命令过滤。比如一个“清理临时文件”工具，可能在插件层面直接调用 `os.remove()`，如果不对插件的权限边界做限制，单靠模型不写危险命令是没用的。

因此需要一种机制，让 Agent 的每一次文件访问都经过一个受控通道，默认情况下写操作不会触碰真实数据。

## 做法/步骤：OpenClaw 的 sandbox 是怎么工作的

OpenClaw 的 sandbox 核心是一个 **overlay/copy-on-write 虚拟文件系统层**。Agent 看到的文件视图并不是宿主文件系统的直接映射，而是在宿主文件系统之上叠加了一层可写快照。当 Agent 执行删除、覆盖、移动等写操作时，这些操作只发生在沙箱层的“影子文件”上，宿主真实文件保持不变。只有通过显式策略允许或人工审批，变更才会合并到真实文件系统。

下面是一个典型的配置与验证步骤。

**1. 启用 sandbox 并设置默认策略**

在 OpenClaw 的 agent 配置中，打开 sandbox，并将默认写权限设为拒绝：

```yaml
sandbox:
  enabled: true
  root_path: /workspace/project
  default_policy: deny-write
  allow_read:
    - /workspace/project/src
    - /workspace/project/config
  allow_write:
    - /workspace/project/tmp
  block_patterns:
    - "rm -rf /"
    - "dd if=/dev/zero"
    - "mkfs"
```

这里的关键是 `default_policy: deny-write`。它意味着所有未显式列出的路径，Agent 只能读，不能写，更不能删除。

**2. 配置高危操作拦截**

除了目录策略，还可以设置命令模式拦截。例如 `rm -rf /`、`rm -rf ~`、`dd if=/dev/zero of=/dev/sda` 这类命令会被 sandbox 直接拒绝，即使策略意外放行，也会被二次拦截。

**3. 使用 dry-run 模式做演练**

在让 Agent 正式执行前，可以先启用 dry-run 模式：

```bash
openclaw run --dry-run task.yaml
```

此时 Agent 会输出它将要执行的文件操作列表，但不实际写入或删除任何宿主文件。这样可以在不产生副作用的情况下检查 Agent 的行为是否符合预期。

**4. 审计日志与人工审批**

所有被 sandbox 拦截的写操作都会记录到审计日志。可以通过以下命令查看：

```bash
openclaw audit show --last 50
```

对于高风险操作，可以配置为“必须人工审批”。审批请求会暂停 Agent 执行，弹出具体操作（哪个进程、哪个路径、什么操作），用户确认后才会放行。

**5. 误删演练**

建议部署后先做一次破坏性测试：故意让 Agent 执行 `rm -rf /workspace/project/important.txt`，观察沙箱是否拦截、日志是否记录、宿主文件是否完好。这一步能验证 sandbox 是否真正生效，而不是只看配置。

## 踩坑点

**把整个根目录或家目录设为可写**

有些用户为了省事，把 `allow_write` 设置为 `/` 或 `/home/user`，这等于让沙箱形同虚设。一旦某个 MCP 插件或命令穿透了虚拟层，就可能直接操作真实文件。

**MCP 插件绕过 sandbox**

不是所有 MCP 插件都走 OpenClaw 的 sandbox 通道。有些插件直接调用宿主 API，或者在插件内部自己实现了文件操作。需要确认插件是否声明了“sandbox-aware”或“受控通道”，否则仅靠框架层配置可能不够。

**审批疲劳**

如果把所有写操作都设置为人工审批，用户会频繁点击“允许”，最终可能养成不看详情就批准的习惯。建议只对删除、覆盖、移动、权限变更等高危操作做人工审批，普通临时文件写入可以让策略自动放行。

**临时文件与性能**

copy-on-write 层在大量小文件读写下会有性能损耗。如果 Agent 需要频繁处理大文件，应考虑把工作目录放在独立的 tmpfs 或临时目录中，避免长期占用沙箱层。

## 可复用建议

- **默认拒绝写操作**：任何 Agent 或插件都不应有隐式写权限。
- **最小权限原则**：只读权限按任务需要显式授予，写权限严格限定在临时目录。
- **显式 allow 优于隐式 deny**：不要让策略有“默认允许”的兜底逻辑。
- **审计日志是最后防线**：定期查看审计日志，发现异常路径访问或非预期写操作。
- **定期做破坏性演练**：模拟误删、误覆盖、路径穿越等攻击，验证 sandbox 是否拦截。
- **把版本控制当作额外保险**：即使 sandbox 被绕过，代码库或数据目录的 Git 快照也能提供恢复能力。

## 总结

OpenClaw 的 sandbox 模型之所以能防止 Agent 误删文件，核心不在于“模型更听话”，而在于它把文件操作从提示词约束变成了运行时强制隔离。默认不可写、写操作重定向到虚拟层、高危命令二次拦截、审计日志记录，这些工程手段共同构成了安全边界。

如果要让 Agent 真正进入生产环境，建议先关闭所有写权限，用 dry-run 模式跑通任务，再逐步放行最小必要的目录，并保留审计日志。安全不是因为相信 Agent 不会犯错，而是因为即使它犯错，破坏也被限制在可回滚的范围内。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/246c7087d177f958.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/411eb26e8332ca84.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/63e567e5e4643dac.png)

