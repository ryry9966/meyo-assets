---
title: OpenClaw 的 AGENTS.md：写给 AI 的工作空间使用手册
feedId: 36906
source: 综合讨论
publishedAt: 2026-09-10
---

## 背景

OpenClaw 的 agent 不只是一个会话窗口，它围绕一个 workspace 运行（默认 `~/.openclaw/workspace`）。这个目录是 agent 的"家"：`SOUL.md` 定义性格，`MEMORY.md` 存长期记忆，`USER.md` 记用户偏好，而 `AGENTS.md` 扮演另一个角色——操作手册。每次新会话启动，它都会被读进上下文，作为 agent 在这个工作空间行动的默认依据。AGENTS.md 本身也是一项社区约定（agents.md），不少编码 agent 都认这个文件名，OpenClaw 沿用了它，并让它承担工作空间级规范。

## 问题：没有手册的 agent 是冷启动的

没有 AGENTS.md 时会发生什么？每次对话都要重新交代：脚本在哪、测试怎么跑、部署走哪条流程、哪些目录不能碰。agent 猜错路径、跑错命令、把临时文件写进正式目录都是常态。更麻烦的是行为漂移：今天它敢直接 `git push`，你纠正一次，隔两天又忘。这些不该由对话来承载——它们是"这个工作空间的规矩"，应该沉淀成文件，而不是留在聊天记录里。

## 做法：把规矩写成可执行的条款

我的 AGENTS.md 稳定在 100 行左右，结构大致是：

```markdown
# 工作空间规范

## 环境
- 主项目路径：~/code/xxx，测试命令：pnpm test
- 部署只走 scripts/deploy.sh，不要手敲 ssh

## 目录约定
- 临时产物放 ./tmp，不污染根目录

## 安全边界
- 禁止 git push -f、rm -rf 绝对路径
- .env 与密钥文件只读；删除超过 10 个文件前必须列出清单确认

## 协作方式
- 中文回复；修改 AGENTS.md 前先给出 diff 征求同意
```

三条写法原则：

1. **写可判定的指令，不写愿望。**"要小心"没有执行语义，"删除前列出清单等待确认"才能被稳定遵守。
2. **分层。**全局规范放 workspace 根目录；项目特有约定（构建命令、目录结构）放到各仓库自己的 AGENTS.md，agent 进入对应目录时读就近那份。
3. **纳入 git。**每次修改走 diff review，防止 agent 自己改手册导致规则漂移。

验证方法很简单：新开会话，问它"当前工作空间的构建和部署约定是什么"，看复述是否准确。

## 踩坑点

- **写太长。**AGENTS.md 每次会话都占上下文，超过三四百行后重要规则会被稀释，token 成本也在涨。手册是压缩过的规范，不是随笔。
- **职责混放。**性格放 SOUL.md，事实记忆放 MEMORY.md，AGENTS.md 只放操作规范。我早期把"用户喜欢简洁回复"写进来，后来发现它属于 SOUL.md。
- **写进敏感信息。**内网地址、API key 一旦写进 AGENTS.md 又推到仓库，就是事故。敏感内容走环境变量或独立密钥文件。
- **只写不验证。**Markdown 没有编译器，规则失效往往在事故后才暴露。定期让 agent"审计"手册：哪些条目过时或互相矛盾，人工确认后再更新。

## 可复用建议

- 用固定骨架起步：环境 → 目录约定 → 安全边界 → 协作方式，四段能覆盖 90% 场景。
- 每条规则尽量带"判定条件 + 动作"，例如"若需删除超过 N 个文件，先列清单等确认"。
- 修改走 PR/diff 流程，让手册和代码一样可回滚。
- 每次在对话里纠正 agent 后，当场把纠正固化进 AGENTS.md，比指望它下次记住有效得多。

## 总结

AGENTS.md 的本质，是把"你对这个工作空间的期望"从对话记忆搬进文件系统，变成每次会话都生效的默认契约。它不玄乎，就是一份给 AI 的 README：短、可执行、有边界、受版本控制。当你发现自己在对话里第三次解释同一条规矩时，就是该写进 AGENTS.md 的信号。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/4346ad0ffdf76202.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/8675b6f34b8235e6.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/d2568f7dfe5dff26.png)

