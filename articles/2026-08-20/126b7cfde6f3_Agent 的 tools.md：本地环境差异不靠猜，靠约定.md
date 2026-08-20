---
title: Agent 的 tools.md：本地环境差异不靠猜，靠约定
feedId: 33919
source: 综合讨论
publishedAt: 2026-08-20
---

在 OpenClaw 或 MCP 插件里让 Agent 自动执行本地命令时，最常见的问题不是 Agent 不会写命令，而是它写的命令在你的机器上跑不通。同样的任务，在同事的 macOS 上成功，在 CI 的 Linux 容器里失败，在你的 Windows 上直接报“command not found”。根源往往不是 Agent 能力不够，而是它缺少一份本地环境的“说明书”。

## 背景

Agent 默认会假设一套标准环境：`python` 在 PATH 里、`ffmpeg` 全局安装、`node` 版本固定。但真实开发环境几乎没人这么干净。有人用 `pyenv`，有人用 `conda`，有人用 `nvm`，还有人把 ffmpeg 装在 `/opt/homebrew/opt/ffmpeg/bin` 下面。Agent 如果只看 `which` 的结果，很容易选到错误版本，或者因为 shell 非交互模式没有加载 `.zshrc` 而找不到路径。

更麻烦的是团队协作：同一个仓库，A 的本地工具路径和 B 不同，CI 又是另一套。如果 Agent 把某个路径写死在生成脚本里，换一台机器就失效。

## 问题

我们过去试过几种做法，都不太理想：

- 在系统提示里写死所有路径：维护困难，换人就得改，而且提示词越来越长。
- 让 Agent 每次自己探测环境：它会执行一堆 `which`、`ls`、`command -v`，既慢又容易误判，还可能触发不必要的副作用。
- 依赖 MCP 工具统一封装：解决了部分问题，但 MCP 服务本身也要配置本地的 token、端口、二进制路径，又绕回来了。

我们需要的是一套轻量、可版本控制、让 Agent 在动手前就能读到的环境约定。

## 做法：用 tools.md 建立本地环境说明书

我们的做法是在项目根部放一个 `tools.md`，或者放在 `~/.openclaw/tools.md` 作为全局兜底。内容只写 Agent 执行本地任务时真正需要的信息，不写废话。

**步骤 1：定义工具清单**

只列 Agent 会高频用到的命令行工具，并为每个工具说明可用版本和来源。示例：

```markdown
## 工具清单

- `python`：使用 `~/.pyenv/shims/python`，默认解释器为 3.11
  - 不要直接调用系统 `/usr/bin/python`
- `ffmpeg`：`/opt/homebrew/opt/ffmpeg/bin/ffmpeg`
  - 如果不存在，先运行 `brew install ffmpeg`
- `node`：通过 `nvm use 20` 激活；CI 中直接用 `node:20-alpine` 镜像
```

**步骤 2：写清路径解析规则**

告诉 Agent 不要相信 PATH，要按顺序检查：

```markdown
## 路径解析规则

1. 优先使用本文件中明确给出的路径
2. 如果路径不存在，使用 `command -v <tool>` 探测
3. 如果探测结果与预期版本不符，停止执行并提示用户，不要自动降级
```

这一步能避免 Agent 找到 Python 2 或系统自带的老版本就硬跑。

**步骤 3：记录环境变量和平台差异**

把需要注入的环境变量、不同平台的差异集中列出：

```markdown
## 环境变量

- `OPENCLAW_HOME`：默认 `/Users/you/.openclaw`，Linux 下为 `/home/you/.openclaw`
- `TMPDIR`：macOS 下使用 `/tmp`，Windows 下使用 `%TEMP%`

## 平台差异

- macOS：Homebrew 路径优先
- Linux：apt 安装路径优先，无 Homebrew
- Windows：所有路径使用 `Get-Command` 探测，不直接使用 `which`
```

**步骤 4：让 Agent 在规划前读取**

在 OpenClaw 的系统提示或 MCP 资源配置里，把 `tools.md` 注册为高优先级上下文。例如：

```yaml
context_files:
  - path: ./tools.md
    priority: high
```

这样 Agent 每次接到本地任务，会先读 `tools.md`，再生成命令。

**步骤 5：维护与校验**

写一个小脚本，列出 `tools.md` 中声明的路径，逐条检查是否存在，避免文档过时。可以把脚本挂到 Git hook 或 CI 里。

## 踩坑点

1. **不要把敏感信息写进 tools.md**。token、私钥、密码一律放环境变量或密钥管理工具，文件本身只写路径和配置名。
2. **不要过度详细**。tools.md 不是所有工具的完整手册，只写 Agent 容易搞错的差异点。太长会影响 Agent 读取效率和上下文窗口。
3. **Windows 路径转义**。在 Markdown 中反斜杠容易被当成转义字符，写 Windows 路径时建议用正斜杠或代码块包裹。
4. **tools.md 与实际环境不同步**。安装新工具后要记得更新，否则 Agent 会读到旧路径然后报错。建议用自动探测脚本生成初版内容，而不是手写全部。
5. **与 MCP 工具冲突**。如果某个能力已经通过 MCP 服务暴露了，本地 CLI 的路径就不用再写进 tools.md，避免 Agent 两套都尝试。

## 可复用建议

- **从模板开始**：准备一份 `tools.md.tpl`，包含工具清单、路径规则、平台差异、已知坑四个段落，新项目直接复制。
- **本地覆盖，版本控制**：全局 `~/.openclaw/tools.md` 放个人机器的通用配置，项目内 `tools.md` 覆盖项目相关工具。两者都纳入版本控制，但全局文件不要提交到项目仓库。
- **与生成脚本结合**：写一个 `generate_tools_md.sh`，自动探测常见工具路径并生成初版，人工只补充特殊差异。
- **定期评审**：每次 Agent 执行失败，先看 `tools.md` 是否缺失了必要信息；把反复出现的问题补进“已知坑”段落。
- **保持克制**：tools.md 的核心价值是“减少猜测”，而不是“替代人类判断”。给 Agent 明确的规则，但允许它在异常时暂停询问。

## 总结

Agent 的本地执行能力上限，不取决于模型多聪明，而取决于它手里有没有准确的环境信息。`tools.md` 不是银弹，但它是目前成本最低、最容易被团队接受的约定方式。把环境差异显式写下来，Agent 才能少踩坑，团队也少一些“我这边跑不了”的来回沟通。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-20/ef7f306525ccb1fe.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-20/5bb3760fb37dbd54.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-20/ae340323b2a9dfe9.png)

