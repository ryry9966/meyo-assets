---
title: Git 自动化实践：让 Agent 帮你管提交和分支，围栏先于效率
feedId: 36332
source: 综合讨论
publishedAt: 2026-09-06
---

## 背景

写了一年代码回头看，Git 里真正花时间的不是解冲突，而是重复动作：想一句像样的 commit message、切分支忘了改名、本地堆了三十个早已合并的 `feat/xxx`。这些事判断含量不高但很烦，正好是 Agent 该干的活。OpenClaw 的 Agent 有工具调用能力，接上 shell 或 MCP 工具后可以直接操作仓库，我最近把这套流程跑通了，记录一下。

## 问题

直接让 Agent“帮我 commit”有三个坑：

1. **它会编**。不看 diff 就写出“优化了部分逻辑”这种废话信息；
2. **它会越权**。一句“清理下分支”可能换来 `git reset --hard`，或对 `main` 的强推；
3. **它会卡住**。Git 触发编辑器、GPG 确认时，Agent 的非交互终端直接挂死。

所以核心问题不是“能不能自动”，而是“在什么围栏里自动”。

## 做法

**第一步：包一层白名单工具。** 不给 Agent 裸 shell，写一个很薄的 Git 工具层（MCP server 或 skill 均可），只暴露 `status / diff / add / commit / branch / log / push` 这类子命令，参数层显式拒绝 `--force`、`reset --hard` 和对保护分支的写操作。围栏做在工具层，比在提示词里求它管用得多。

**第二步：固定提交流程。** 在 skill 的系统提示里写死步骤：先 `git status` 和分块 `git diff`，基于真实变更生成 Conventional Commits 风格的信息，输出“计划 + message”等人确认，再执行 `add` 和 `commit`。关键是强制“先看 diff 再写字”，这一条能挡掉大部分幻觉提交。

**第三步：分支治理做成定时任务。** 每周跑一次：`git branch --merged` 对照远端，列出可删分支生成清单，人工确认后批量删除。清理是建议式的，删除永远过一道人。

**第四步：留审计日志。** 每次工具调用记录命令、参数、结果摘要，出问题能回放。

## 踩坑点

- **大 diff 撑爆上下文**：一次改二十个文件时，让 Agent 分块总结再合成 message，别一次性塞进去；
- **交互卡死**：用 `git -c core.editor=true commit ...` 绕开编辑器；GPG 签名环境要在 credential helper 层提前配好，别指望 Agent 现场处理；
- **并发撞锁**：两个会话同时操作同一仓库会撞 `index.lock`，简单解法是单实例排队，别上来就搞复杂的锁方案；
- **凭据暴露**：Agent 能执行命令就能读 `.git/config`，远端凭据走 ssh-agent，不要把 token 明文给到上下文。

## 可复用建议

- 把“提交规范”写进 skill 的系统提示，而不是每次对话重复；
- 所有破坏性操作先 dry-run 输出计划，确认后再执行；
- 工具层白名单 + 人工确认这两道闸，是这类自动化的底线配置，别为省一步拆掉；
- 审计日志从第一天就开，成本极低，回溯时救命。

## 总结

Agent 适合接管 Git 中“重复但有判断”的部分：message 生成、变更摘要、分支清理建议。危险操作留在人手里，自动化程度随信任积累逐步放开。这套东西一个下午能搭完，收益是每天省掉十几分钟琐碎操作，值得做。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/4f49d263490a11d4.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/bcce2de6aeb6dfb0.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/605853a2ae7ef93b.png)

