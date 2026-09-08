---
title: 让 Agent 替你管 Git：一套可控的提交与分支自动化实践
feedId: 36602
source: 综合讨论
publishedAt: 2026-09-08
---

## 背景

写得好的 commit message、干净的分支结构，价值大家都认，但很少有人愿意每天手动维护。接入 OpenClaw 之后，agent 本身具备执行 shell、调用 MCP 工具的能力，把 Git 这种高频、规则明确的操作交给它，是收益最直接的场景之一。

我在自己的工作流里跑了一个多月，目标就三件事：提交信息规范化、分支命名与清理、未提交改动的定时提醒。下面把做法和踩的坑整理出来。

## 问题

直接让 agent「帮我 commit」有几个明显问题：

1. **输出不可控**：不加约束时，生成的 message 经常是 `update code` 这种无信息量描述，或者自由发挥，写进不属于本次改动的内容。
2. **权限过大**：`git add -A`、`git push --force`、对 main 直接操作，一旦放开都可能出事故。
3. **上下文限制**：大 diff 塞进上下文，要么截断导致描述失真，要么直接超预算。

## 做法

### 1. 先做工具层收权，再谈自动化

我没有暴露原始 shell，而是写了个白名单包装脚本作为工具（MCP 工具或 skill 内调用均可）：

```bash
#!/bin/bash
# safe-git.sh：白名单化 git 入口
case "$1" in
  status)  git status --short ;;
  diff)    git diff --staged ;;
  add)     git add "$2" ;;
  commit)  git commit -m "$2" ;;
  branch)  git branch "$2" ;;
  log)     git log --oneline -10 ;;
  *)       echo "denied"; exit 1 ;;
esac
```

push 单独留入口，但只允许当前分支推到同名远端分支，且显式禁止 main：

```bash
push)
  current=$(git branch --show-current)
  if [ "$current" = "main" ]; then echo "denied: protected branch"; exit 1; fi
  git push origin "$current" ;;
```

### 2. 提交流程固定为三步

在 skill 指令里写死流程，不让 agent 自由决策：

- 先 `status` + `diff --staged`，确认 staged 内容非空；
- **基于实际 diff**（不是用户口头描述）生成 Conventional Commits 格式的 message，含 type 和 scope；
- 输出 message 和待提交文件列表，等我确认后再执行 commit。

确认这一步不能省。它把 AI 从「执行者」降级为「起草者」，风险立刻可控。

### 3. 分支管理规则化

- 命名模板：`feat/`、`fix/`、`chore/` 前缀 + 短横线描述，agent 创建前先校验格式；
- 每天 heartbeat 定时跑一次 `git branch --merged` 加最后提交时间，汇总已合并可删分支和超过 14 天未动的分支，输出清单发给我——删不删我来定。

## 踩坑点

- **diff 太大**：几百行的 diff 会让 agent 的描述开始泛化。改成先 `diff --stat` 看规模，超阈值就按文件分批总结再合并。
- **AI 总结的是意图不是事实**：有次我口头说「修了个登录 bug」，agent 把这句写进 message，但 staged 里其实还混着半成品重构。规则改成「禁止参考对话内容，只允许 diff」后才稳定。
- **误加敏感文件**：在 `add` 入口加了检查，命中 `.env`、`*.pem`、密钥目录直接拒绝，列出来人工处理。
- **pre-commit hook 修改文件**：格式化 hook 改完文件后，agent 以为提交完成，工作区其实还是脏的。流程里 commit 之后必须再跑一次 `status` 确认。
- **rebase/冲突不要交给 agent**：试过让它自动解决冲突，结果它「合理地」丢了一段代码。现在遇到冲突一律停下报告，人来处理。

## 可复用建议

1. 工具层白名单 > prompt 里写「不要做什么」，约束放在代码里才可靠。
2. 所有 agent 的 git 操作落一份日志（时间、命令、参数），出问题能回溯。
3. 自动化只做「低风险 + 可逆」的操作：commit 可 reset、建分支可删；push、rebase、clean 保持人工确认。
4. 定时任务输出**清单**而不是直接执行清理，决策权留在人这里。
5. message 规则越具体越好：限定格式、限定信息来源、限定长度，比一句「写好一点」有用得多。

## 总结

这套东西没有任何黑科技，就是白名单脚本 + 固定流程 + 确认门 + 定时报告。一个月下来，commit 历史第一次做到了整齐一致，长期滞留的分支也清掉了。经验一句话：**让 agent 做「起草和巡检」，让人保留「确认和删除」的权限**，Git 自动化基本就不会翻车。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-08/89534f18a2d6df80.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-08/2b8403d5ebc52426.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-08/94196021e31d1ba5.png)

