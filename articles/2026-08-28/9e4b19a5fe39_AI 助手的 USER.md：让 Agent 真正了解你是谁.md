---
title: AI 助手的 USER.md：让 Agent 真正了解你是谁
feedId: 35094
source: 综合讨论
publishedAt: 2026-08-28
---

## 背景

在 OpenClaw 这类 Agent 工作台里，我们习惯把时间花在 MCP 工具接入、插件权限、自动化脚本上，却容易忽略一个更基础的问题：Agent 每次启动时，其实并不认识你。

它不知道你的机器是 macOS 还是 Linux，不知道你默认用 pnpm 还是 npm，不知道你的测试命令是 `pytest -q` 还是 `go test ./...`。于是每次执行任务，它要么反复问，要么按一个“平均用户”的习惯猜。猜错时，轻则重新生成脚本，重则跑到错误路径、用错包管理器，甚至碰了不该碰的目录。

USER.md 要解决的就是这件事：把关于“你”的稳定事实，放到一个 Agent 可读取的文本文件里，让它不再从零开始。

## 问题：上下文缺失的典型表现

如果你在 OpenClaw 或类似 Agent 环境里经常遇到下面情况，就说明缺少用户级上下文：

- 每次任务都问“你是什么系统”“用哪个包管理器”；
- 生成的脚本硬编码 `/home/user`，但你的环境是 macOS；
- 在非交互环境里动不动就 `sudo`；
- 自动改配置时选错路径，例如在 `~/.config` 和 `~/.local` 之间反复横跳；
- 输出格式、注释语言、commit message 风格每次都不一样。

这些大多不是模型能力问题，而是它没有拿到稳定、可复用的用户事实。

## 做法：把“你是谁”工程化

### 1. 先确认加载位置

全局用户级信息可以放在 `~/.openclaw/USER.md`；项目级信息可以放在 `.openclaw/USER.md`。不同版本或启动器可能不会自动加载，所以在 OpenClaw 的初始化提示里显式加一句：

```text
Before planning, read ~/.openclaw/USER.md if it exists; otherwise ask me to create a minimal one.
```

如果你的 OpenClaw 配置不支持自定义初始化提示，可以把 USER.md 当作普通资源，通过 MCP 的 filesystem 工具让 Agent 主动读取。

### 2. 用最小结构组织内容

一个可维护的 USER.md 建议控制在 80-120 行以内，按“身份、环境事实、工作偏好、约束”四块组织。

```markdown
# USER.md

## 身份
- 后端 / 基础设施工程师，常用 Go、Python、Shell
- 机器：macOS 14，Apple Silicon
- Shell：zsh；默认包管理：Homebrew + pnpm

## 环境事实
- 工作目录：~/work
- 常用测试：go test ./...、pytest -q
- 无 GPU；不要建议 CUDA 相关方案

## 工作偏好
- 注释和 commit message 使用中文
- 命令优先使用非交互模式
- 输出日志级别：info

## 约束
- 不要修改 ~/.ssh、/etc
- 不要运行 rm -rf 或包含危险通配符的命令
- 敏感信息从环境变量读取，不要写进脚本
```

### 3. 让 Agent 在任务开始前核对

更好的做法是要求 Agent 开场先读取 USER.md，并输出一个简短的“已知环境摘要”。你可以把它写到初始化提示里：

```text
Read USER.md, then print a 3-line summary of OS, shell, default package manager, and hard constraints before continuing.
```

这样既确认 Agent 真的读了，也能让人快速发现信息是否过时。

## 踩坑点

1. **写太长**：超过 120 行后，Agent 会把关键约束淹没在细节里。全局 USER.md 只放稳定事实，项目级放项目约束。
2. **把秘密写进去**：token、私钥、密码、API key 绝不要写进 USER.md。它们应该走环境变量或 secret manager。USER.md 可被日志、截图、共享会话带走。
3. **路径硬编码**：换机器后 `~/work/src/foo` 可能不存在。优先写 `~/work` 或 `$HOME`，并标注“以下路径为当前机器”。
4. **信息过期**：加 `updated` 字段，并让 Agent 在开场用 `uname -a`、`node -v`、`pnpm -v` 快速核对。发现不一致就提醒你更新。
5. **与系统安全策略冲突**：不要试图用 USER.md 覆盖 OpenClaw 的 allowlist/denylist 或插件权限。安全策略优先于用户偏好。

## 可复用建议

- 采用“事实-偏好-约束”三段式，每行一条，避免长段落。
- 事实项尽量可验证：写“macOS 14”而不是“较新的苹果系统”。
- 如果团队共用，把 USER.md 纳入版本控制，但剥离个人敏感信息，保留团队基线。
- 配合 MCP 的 memory/knowledge 类工具做动态更新，但不要用动态记忆代替静态 USER.md。
- 像维护代码一样定期评审 USER.md，删除不再适用的条目。

## 总结

USER.md 不是提示词魔法，也没有复杂格式。它只是把“为谁、在哪、按什么规则做”这些长期稳定的信息外部化。OpenClaw 里的 MCP 和插件决定 Agent 能做什么，USER.md 决定它是否做得像“你”。先把这一层补上，很多重复提问和低效猜测会自然消失。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/c0daa978ed93ac88.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/7b8b02785278d193.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/1614475e5d7bbf26.png)

