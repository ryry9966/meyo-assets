---
title: 让 Agent 不再猜你：OpenClaw 场景下的 USER.md 实践
feedId: 35502
source: 综合讨论
publishedAt: 2026-08-31
---

## 背景：上下文里最缺的一块

在 OpenClaw、多 Agent、MCP、插件和自动化实践里，我们花了大量时间写 system prompt、工具描述、MCP tool schema、插件配置，却很少把“用户是谁”当作一等上下文。

结果很常见：Agent 反复问你同样的问题——用哪个 Python、装什么包管理器、提交信息用什么格式、能不能动 lockfile、要不要写测试、输出中文还是英文。每次切换项目或换机器，都要重新解释一遍。

USER.md 就是解决这类问题的低成本文件。它不是自我介绍，也不是日志，而是 Agent 在运行前可以读取的一份“用户运行环境与约束说明”。

## 问题：没有 USER.md 或写得太泛

如果完全不提供用户上下文，Agent 只能靠猜。它可能用错 Node 版本、全局安装依赖、生成不符合团队规范的提交信息、把你的实验目录当成正式项目目录、在你不允许联网时尝试调外部 API。

但写得不好也常见：

- 写成散文，Agent 提取不到可执行信息；
- 把 token、密码、内部地址放进去，产生泄露风险；
- 信息过期，Agent 比没有上下文时错得更自信；
- 与 system prompt 冲突，模型不知道该听谁。

所以 USER.md 的关键不是“写得多”，而是“稳定、可执行、低敏感、能更新”。

## 做法：一份可用的 USER.md 应该长什么样

### 1. 只放稳定、影响决策的信息

USER.md 不应包含临时任务状态、当次调试过程、当前进度。它是相对稳定的用户画像和项目约束。任务状态应该放在 task/state 文件里。

### 2. 推荐结构

```markdown
# USER.md

## 环境
- OS: macOS / Linux
- Shell: zsh
- Editor: Neovim
- Timezone: Asia/Shanghai
- Language: 中文回复，代码与提交信息用英文

## 项目约束
- 包管理：pnpm，不要改 package-lock.json
- Node: 20
- Python: 3.12，使用 venv，不全局 pip install
- 测试：pytest / vitest
- 格式化：提交前跑 lint + format

## 命名与目录
- 组件文件：kebab-case
- 测试文件：*.test.ts / test_*.py
- 文档放 docs/

## 输出偏好
- 先给结论，再给步骤
- 命令默认加解释，不要直接跑危险操作
- 提交信息遵循 conventional commits

## 禁止事项
- 不要主动创建 README、CHANGELOG
- 不要修改 CI 配置
- 不要访问网络以下载未声明依赖
- 不要用 sudo
```

### 3. 接入 OpenClaw/Agent

OpenClaw 场景下，有几种接入方式：

- **作为工作区文件**：把 USER.md 放在项目根目录或 home 目录，让 Agent 启动时读取。
- **注入 system prompt**：在 OpenClaw 的 session/插件配置里，把 USER.md 内容拼到 system prompt 尾部。
- **通过 MCP 暴露**：写一个很小的 MCP resource/tool，让模型按需读取用户偏好，而不是每次全量注入。
- **版本管理**：放进 dotfiles 仓库，修改走 PR，避免随手改完不记得改了什么。

### 4. 写成“可执行条目”，而不是自然语言段落

好的 USER.md 更像配置文件，不像日记。多用列表、开关、命令示例，少用“我比较喜欢干净整洁的代码”这种不可执行的话。

## 踩坑点

- **太长**：超过 150 行后，Agent 容易忽略或上下文截断。控制在 100–150 行以内。
- **放敏感信息**：不要把 token、密码、内网地址写进去。需要用密钥时，引用环境变量或 secret manager。
- **与 system prompt 冲突**：比如 system prompt 说“尽量详细解释”，USER.md 说“只给结论”，模型可能无所适从。可以明确写“如与 system prompt 冲突，以 system prompt 为主”。
- **写入临时状态**：USER.md 不是任务清单。出现“当前在做 XX”这样的内容，说明放错文件了。
- **信息过期**：环境、版本、路径变了但没更新，Agent 会按照旧环境操作，比没有 USER.md 更有害。
- **绝对化偏好**：不要写“永远不要道歉”“永远不要解释”这类绝对句，容易和模型内置策略冲突。写成“优先避免道歉”“默认不展开解释”更好。

## 可复用建议

- 把 USER.md 分层：**全局用户偏好**放 home/dotfiles，**项目约束**放仓库。不要把所有个人偏好都塞进团队仓库。
- 建立最小读取流程：每次新会话开始，让 Agent 先读 USER.md；如果缺失，只问最少必要问题。
- 用同一任务对比测试：有 USER.md 和没有 USER.md 各跑一次，统计 Agent 的提问数量和错误命令，能明显看出价值。
- 不要让它变成“万能上下文”：它替代不了 system prompt、项目文档和工具 schema。它的定位是补充用户侧约束，不是包罗万象。

## 总结

USER.md 是一个很小但杠杆很高的上下文资产。对 OpenClaw/Agent/MCP 实践者来说，它的核心价值是减少重复确认、避免环境误判、让自动化更稳定。

写好它的原则可以压缩成四句：

1. 只放稳定信息；
2. 写成可执行条目；
3. 不碰秘密和临时状态；
4. 定期更新，冲突时明确优先级。

如果你已经维护了 system prompt 和 MCP 配置，却总觉得 Agent“不够了解你”，下一份该补的文件很可能就是 USER.md。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/61cb7e81bf51665b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/04e9a4d5269c6e75.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/0f82b124e051c9d6.png)

