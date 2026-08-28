---
title: AI 助手的 USER.md：让 Agent 真正了解你是谁
feedId: 35126
source: 综合讨论
publishedAt: 2026-08-29
---

# AI 助手的 USER.md：让 Agent 真正了解你是谁

## 背景

在 OpenClaw、MCP 和插件自动化场景里，我们希望 agent 不只是执行单次指令，而是能按你的习惯工作：用哪个 shell、代码风格、目录结构、哪些命令禁止、哪些 MCP 工具优先。现实是，这些信息散落在每次对话里，换会话就丢。对个人用户来说，最轻量可控的落地方式不是再上一套复杂记忆系统，而是维护一个 `USER.md`。

## 问题

没有 `USER.md` 时，常见现象：

- agent 默认用 `python` 而不是 `python3`，或执行了高风险命令；
- 每次新任务都要重复“我的环境是……”“不要动某个目录”；
- agent 调用 MCP 工具时选择很随机，缺少个人偏好约束；
- 聊天记录越堆越长，看似有记忆，但不可复用、不可审计。

`USER.md` 不是万能记忆，而是一份稳定、短小、可加载的个人运行上下文。

## 做法 / 步骤

### 1. 控制边界：只写稳定信息

`USER.md` 只放三类内容：

- **事实**：OS、常用路径、时区、编辑器、主力语言、硬件限制；
- **偏好**：命令风格、输出语言、commit 风格、日志级别、是否自动确认；
- **禁止**：不允许删除、不允许访问的目录、不允许联网查询的域名、不允许在未备份前修改的文件。

敏感信息不要写入，包括 token、密码、私钥、cookie。

### 2. 选择加载位置

在 OpenClaw 里，建议建两个层级：

- 全局：`~/.openclaw/USER.md`，跨项目不变的偏好；
- 项目：`.openclaw/USER.md`，项目相关路径、命令、MCP 工具选择。

然后让 agent 在冷启动或首次执行前读取。不同执行器的接入点不同：有的是 system prompt 静态注入，有的是通过 MCP resource 暴露，有的需要插件在会话创建时读取。选择哪种取决于上下文窗口大小和隐私要求。

### 3. 写一个可执行模板

```markdown
# USER.md

## 事实
- OS: macOS 14 / Debian 12
- Shell: zsh
- Editor: nvim
- Timezone: Asia/Shanghai
- Python: 优先 python3.11

## 偏好
- 输出使用简体中文，技术术语保留英文
- 命令先给 dry-run，再执行
- 文件操作前先备份到 /tmp/oc-backup/

## 禁止
- 不执行 rm -rf
- 不读取 ~/.ssh/ 下任何文件
- 不访问生产环境数据库，除非显式要求并提供连接标识
```

控制在 150 行以内。每一行都应该能在某个任务里直接影响 agent 的下一步动作。太像自我介绍但没约束力的内容可以删。

### 4. 拆分配置，按需加载

不要每次把整份 `USER.md` 全部塞进 prompt。可以拆成：

- `profile.md`：环境事实；
- `preferences.md`：工作偏好；
- `constraints.md`：安全边界。

首次会话只加载 profile 的摘要，agent 遇到相关动作时再读取对应文件。这样既省 token，也避免长上下文后半段被忽略。

### 5. 建立更新闭环

每两周或每次踩坑后回看：哪些信息 agent 反复问？哪些偏好它没遵守？把重复澄清的内容写回 `USER.md`。一个简单方法是在 agent 配置里加入提醒：任务结束前，若有信息值得沉淀，建议追加到 `USER.md`。

## 踩坑点

- **把 USER.md 当博客写**：写到 500 行，agent 实际只读了前 80 行。
- **信息矛盾**：全局 `USER.md` 和项目 `AGENTS.md` 同时定义了 python 版本，agent 按顺序取了错误的一个。需要明确优先级，例如项目级 > 全局。
- **泄漏风险**：通过 MCP filesystem 工具读取 `USER.md` 时，可能被第三方 MCP server 记录完整路径；不要在文件里放敏感字段。
- **环境漂移**：换机器后 `USER.md` 里的路径还是旧 Mac 的，agent 在 Linux 上找不到，报错后可能自行猜测补全，导致更糟结果。
- **过度约束**：把“不要解释太多”写成“少说话”，agent 可能在需要确认风险时也直接执行。
- **误提交敏感内容**：把 `USER.md` 写进项目提交但忘记排除敏感项。建议用 env var 注入敏感值，文件本身只写占位符。

## 可复用建议

- 用“事实-偏好-禁止”三段式，每条不超过两行。
- 给 agent 一个固定加载入口：新会话先读 `USER.md`，再开始任务。
- 在 `USER.md` 顶部写优先级说明：本文件优先级高于默认行为，低于项目 `AGENTS.md`/显式指令。
- 做一次冒烟测试：新环境里问 agent “我的默认 shell 是什么？有哪些禁止操作？” 如果答错，说明加载失败或文件被截断。
- 把 `USER.md` 纳入配置管理，但不要放 secrets；敏感项用 `${ENV_VAR}` 占位，由启动脚本注入。
- 如果你的 MCP 支持 resource，优先用 resource 暴露 `USER.md`，而不是在每次 system prompt 里重复粘贴。

## 总结

`USER.md` 的价值不在“让 agent 更像你”，而在减少重复澄清和不一致操作。它应该是一个小型的、可复现的运行配置：稳定、精简、有明确优先级、有更新机制。对 OpenClaw/Agent/MCP 用户来说，这比堆更多插件或更复杂的 memory 系统更可控，也更适合长期维护。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/74e5541acf7913ff.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/83fde889556e41d3.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/833da45ec7ac0ae6.png)

