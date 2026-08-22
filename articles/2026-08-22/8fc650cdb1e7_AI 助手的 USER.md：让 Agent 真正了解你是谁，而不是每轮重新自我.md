---
title: AI 助手的 USER.md：让 Agent 真正了解你是谁，而不是每轮重新自我介绍
feedId: 34193
source: 综合讨论
publishedAt: 2026-08-22
---

# AI 助手的 USER.md：让 Agent 真正了解你是谁

## 背景

做 OpenClaw、Agent、MCP 相关自动化的时间越久，越容易发现一个问题：项目级上下文（如 AGENTS.md）只管仓库规范，不管“使用工具的人是谁”。于是每次新会话开始，你都要重复交代环境、偏好、禁忌。说得少，Agent 会按通用方式干活，容易在包管理器、目录结构、提交习惯上踩坑；说得多，又浪费上下文，还经常遗漏。

USER.md 的价值就在这里：它是一份用户级说明书，把静态的、可复用的“你是谁”沉淀下来，让 Agent 在进入项目之前先加载，减少重复沟通。

## 问题

默认 Agent 的行为是通用化的。它不知道你更习惯用 pnpm 还是 npm，不知道你在 macOS 上跑 OrbStack，不知道你讨厌自动 push。很多自动化翻车不是因为能力不够，而是因为缺少个人上下文。更麻烦的是，敏感信息如果靠人肉粘贴，既容易泄漏，也不稳定。

USER.md 解决的不是“让 Agent 变聪明”，而是“让 Agent 少做错误假设”。

## 做法/步骤

1. **确定加载位置**。OpenClaw 一般支持用户级目录下的 `USER.md`，例如 `~/.openclaw/USER.md`。如果当前版本没有自动注入，可以在 Agent 启动配置或全局 prompt 里显式引用。项目规则继续放在项目的 `AGENTS.md` 中，两者分工。

2. **按模板写，别写散文**。推荐最小结构：

```markdown
# USER.md

## 身份
- 后端/基础设施方向，主要语言 Go、Python
- 常用 macOS / Linux 环境

## 环境与工具
- 默认 shell: zsh
- 包管理: mise 管理 Python/Node，pnpm 优先于 npm
- 常用路径: ~/src, ~/lab

## 偏好
- 先给可执行方案，再给代码
- 注释与提交信息使用英文
- 优先标准库，避免为小功能引入重依赖

## 约束
- 不要自动执行 git push 或破坏性命令
- 不要修改 ~/.ssh、~/.aws 下的文件
- 遇到不确定的依赖升级先询问
```

3. **保持每条可执行**。像“写得好一点”这类偏好没有用，要写成“先给方案，再给代码”“提交信息遵循 Conventional Commits”。

4. **只放静态信息**。动态状态（比如当前分支、待办清单）交给 MCP 工具或插件去取，USER.md 只放稳定偏好和索引。密钥、token 一律不放。

5. **纳入版本管理**。把 USER.md 放进 dotfiles 仓库，像维护 README 一样定期裁剪。

## 踩坑点

- **内容过载**：USER.md 太长，每次会话都注入，浪费 token。建议控制在 80~150 行以内。
- **写入敏感信息**：USER.md 可能被会话记录、日志或 Agent 输出引用，不要放密码、API key、私钥路径的完整内容。
- **偏好模糊**：写成“代码质量高一点”，Agent 无法执行。要给出可判断的标准。
- **和项目规则冲突**：项目规则和用户规则不一致时，Agent 可能摇摆。建议明确优先级：项目 AGENTS.md 优先于 USER.md，冲突时以项目为准。
- **旧会话不生效**：修改 USER.md 后，已经启动的 Agent 不会自动重新加载，需要新会话或显式 reload。
- **共用环境串味**：如果同一台机器多人使用同一个 home，USER.md 会互相污染。可以用环境变量或目录隔离。

## 可复用建议

- 做一个“快速版”和“完整版”：日常用 10 行以内的身份 + 偏好 + 禁忌，特殊项目再补充。
- 把 USER.md 当成接口，而不是日志。每三个月清理一次过期的工具、路径、偏好。
- 如果 OpenClaw 支持 MCP，把个人数据做成 MCP server 返回，USER.md 只写“获取个人信息用 xxx 工具”。
- 配合插件自动化时，明确权限边界，比如“可以改 ~/src 下的文件，但不要碰系统配置”。
- 考虑团队场景：团队级规范放 AGENTS.md，个人级习惯放 USER.md，避免混在一起。

## 总结

USER.md 是典型低成本、高回报的 Agent 配置。它不会让 Agent 突然变强，但能让 Agent 少犯个人假设错误，把重复的自我介绍从每轮对话里拿掉。写好它的关键不是堆内容，而是结构化、可执行、持续维护。对 OpenClaw 用户来说，这比多装几个插件更值得先做。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/4a6dad6909d374fa.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/c4aa0803498f5f65.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/28b897a8601c1843.png)

