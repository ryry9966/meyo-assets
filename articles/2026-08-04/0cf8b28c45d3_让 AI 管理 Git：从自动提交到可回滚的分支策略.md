---
title: 让 AI 管理 Git：从自动提交到可回滚的分支策略
feedId: 31574
source: 综合讨论
publishedAt: 2026-08-04
---

去年底，我们团队开始把 OpenClaw 接入日常开发流。AI 生成代码已经不算新鲜事，真正的拐点是让它直接操作 Git——自动 commit、自动切分支、自动提 PR。

刚开始很爽。但两周后，我们几乎回滚了三次仓库，丢掉过一份未备份的环境变量文件，还在 develop 分支上留下了一串 “update” 开头的垃圾提交信息。今天这篇帖，想把这些踩过的坑和最终沉淀下来的做法讲清楚。

### 为什么 AI 管 Git 会出事

不是 AI 蠢，是边界不清晰。

人操作 Git 有隐性纪律：知道当前在哪个分支、知道哪些文件是生成物不该提交、知道 commit message 要写清上下文。AI 没有这个直觉。它只看到了指令——比如“提交一下今天的改动”——然后它会把工作区所有变更一股脑 `git add . && git commit -m "update"`，甚至顺手 `git push --force` 到远端。

更麻烦的是，OpenClaw 这类 Agent 天然有“自主完成任务”的倾向。你让它修 bug，它就真的会把修改、测试、提交、推送一条龙做完。速度快，但不可控。

### 我们的做法：给 AI 装三套保险

**第一套：隔离环境，禁止裸奔**

给每个 Agent 任务分配独立的沙盒工作目录，目录名一律用 `workspace_<task_id>`。这样做的好处是：就算 AI 在目录里胡搞，也不影响主 repo。另外在 MCP 配置里只暴露当前任务相关的仓库路径，从源头屏蔽 AI 访问其他项目的可能。

**第二套：收窄指令集，封装危险操作**

OpenClaw 的 MCP 工具里默认有完整的 Git 能力，但我们把 `reset --hard`、`push --force`、`rebase` 这几个高阶操作从工具列表里移除了。取而代之的是自定义封装脚本：

```bash
function hard_reset() {
    echo "This operation is not allowed in agent mode."
    exit 1
}
```

同时，我们配置了允许白名单：

```json
{
  "allowed_tools": ["git_add", "git_commit", "git_branch", "git_checkout", "git_merge", "git_status"]
}
```

**第三套：commit 前强制 diff review**

这是最有效的一条规则：AI 遇到任何需要提交的场景，都必须先执行 `git diff` 并将结果输出到一个 review 文件，等待人工确认，然后才能在 `AGENTS.md` 的约定下执行 commit。我们把这条写进了 OpenClaw 的 skill 定义里。

模板长这样：

```markdown
## 提交规则
- 绝不执行 git push 除非收到显式 push 指令
- commit message 遵循 Conventional Commits 格式（feat/fix/docs/refactor...）
- 提交前必须运行 lint 和测试
- 每 15 分钟内最多执行 1 次 commit，防止连击
```

### 踩过的坑

**坑一：AI 在错误分支上造 PR**

有一次 Agent 在 `fix/oauth-timeout` 分支上开发，完成时检查发现远端 `main` 已经落后，它居然想 `git pull origin main` 然后自动 merge。结果合并了别人的半成品代码，CI 直接红灯。事后我们把 `git pull` 也设成了需要双重确认的操作。

**坑二：commit 整夜挂起**

某次 AI 执行 commit 时，进程一直挂起，卡了整晚。排查后原因是本机配置了 GPG 签名，但 Agent 调用的 Git 工具没有传 `-S` flag，弹出了交互式密码输入。解决方式很简单：在沙盒环境里禁用交互式提示，统一声明 `GPG_TTY=/dev/null`。

**坑三：提交信息看似规范，实质无效**

AI 学会了写 Conventional Commit 格式，但内容变成了：

```
fix: 修复一些变量命名问题
```

这个信息对代码 review 毫无价值。后来我们强制在 commit message 里附加关联的 task_id 和变更摘要，让提交信息可以被追踪。

**坑四：并发操作锁冲突**

多任务并行时，两个 Agent 同时改同一个文件，导致 Git index.lock 冲突。后来我们规定每个 Agent 必须在独立分支工作，合并全部走 PR，禁止 Agent 直接 merge 到集成分支。

### 可复用的三条建议

1. **做一个 `AGENTS.md` 放在仓库根目录**，让 OpenClaw 在进入目录时强制读取。把提交规则、分支命名规范、CI 要求都写进去。AI 会认真读，因为它知道不遵守就会被人工核毙。

2. **用 Git 自身能力做保险**。开启 branch protection，要求 PR review 通过后才允许 merge。这样即使 AI 手滑 push 了，也无法直接破坏主干。

3. **定期检查 reflog**。宁可让 AI 多犯错，也不要跑过度严格的 hook 把 Agent 卡死。每周末人工扫一遍 reflog，发现异常就回滚。

### 总结

让 AI 管理 Git 的正确姿势，不是追求让它全自动，而是让它在一个明确受限的边界内自主行动。

我们的最终清单是：

- 隔离环境 + 白名单工具集
- commit 前 diff review + 强制提交规范
- 分支独立 + PR 合并 + 人工最终审批

这套机制运行了一个月，没有再发生意外覆盖或大规模回滚。AI 仍然负责自动提交、批量处理，但每一次变更都有人兜底。

真正的工程化，是给工具立好边界，而不是依赖工具本身的自觉。

---

