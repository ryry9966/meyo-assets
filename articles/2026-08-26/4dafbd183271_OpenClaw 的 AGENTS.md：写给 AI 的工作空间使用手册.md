---
title: OpenClaw 的 AGENTS.md：写给 AI 的工作空间使用手册
feedId: 34748
source: 综合讨论
publishedAt: 2026-08-26
---

## 背景

OpenClaw 把 Agent 直接放进工作空间后，仓库里最难传递的不是代码，而是那些默认约定：哪些命令可以跑、哪些目录不能动、MCP 工具应该怎么选、插件触发前要检查什么。这些事情人看一眼 README 或团队 wiki 就能理解，但 Agent 不会自动获得这些上下文。AGENTS.md 就是在这层做的补充：它不是给人看的项目介绍，而是给 AI 的操作边界和使用手册。

## 问题

没有 AGENTS.md 时，OpenClaw 的 Agent 通常会出现三类行为漂移。第一，命令靠猜。你让它“跑一下测试”，它可能执行 `pytest`、`npm test` 或自己编一个 `make test`，取决于模型对项目类型的惯性判断。第二，工具选择不稳定。同一个任务里，filesystem MCP、git MCP、插件脚本可能都能完成，但 Agent 会优先选最顺手的，而不是仓库希望它用的。第三，重复踩坑。多人或多次会话中，Agent 反复犯同一个错误，因为没有地方把“这里必须走 scripts/check.sh”固化下来。

## 做法/步骤

第一步，先盘点 Agent 真正需要知道什么。建议只写五类信息：项目入口与目录结构、常用命令、工具/MCP 使用策略、禁止事项、验证方式。业务背景不用写太多，Agent 需要的是可执行信息。

第二步，在仓库根目录创建 `AGENTS.md`。如果 OpenClaw 支持自动加载，确认它读取的是根目录还是 `.openclaw/` 下；如果不行，就在任务模板或会话开头明确要求“先读 AGENTS.md”。

第三步，内容尽量写成短指令。比如：

```markdown
# AGENTS.md

## 常用命令
- 后端测试：`cd backend && make test`
- 前端检查：`cd frontend && npm run lint`
- 全量验证：`./scripts/ci-check.sh`

## MCP 工具策略
- 文件读写优先使用 filesystem MCP
- 浏览器自动化只在 e2e/ 目录任务中使用
- 不要用 git MCP 执行 force push 或修改历史

## 边界
- 禁止读取 .env、上传密钥
- 不要修改 infra/terraform/*.tf 的 state 配置
- 依赖大版本升级必须先人工确认
```

第四步，让验证路径闭环。可以在 AGENTS.md 末尾写明“任务完成后运行 `./scripts/ci-check.sh`，并把失败信息带回”。

## 踩坑点

第一个坑是把 AGENTS.md 写成 README 副本。大段业务背景、架构演进历史对 Agent 没有用处，还会占用上下文。第二个坑是命令写得太抽象，比如只写“运行测试”，Agent 仍然会自行发挥。第三个坑是文件过长。AGENTS.md 每轮都可能被注入，超过 500 行会明显消耗 token，建议把详细 runbook 放到 `docs/agent/` 下，AGENTS.md 只保留指针。第四个坑是信息过期。命令变了但 AGENTS.md 没改，Agent 会严格按旧命令执行，比没有说明更危险。第五个坑是边界只写“注意安全”，不如直接写“禁止执行 rm -rf”“禁止读取 .env”。

## 可复用建议

- 建立模板：把通用策略抽成 `AGENTS.template.md`，新项目复制后只改目录和命令。
- 保持单一事实来源：命令说明只出现在 AGENTS.md 或 scripts 注释里，不要同时维护多份。
- 让 Agent 反向反馈：任务结束时让它标记哪些说明不清晰，定期修订。
- 不要放敏感信息：AGENTS.md 会被模型读取，可能进入日志或第三方工具，密钥、内部地址尽量用变量引用。

## 总结

AGENTS.md 的价值不在于写得多全，而在于把 Agent 的高频不确定性降下来。它更像工作空间的“操作边界”，而不是项目文档。内容克制、可执行、可持续维护，才能让 OpenClaw 的 Agent 在多次会话和多人协作中保持行为稳定。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/aecd9fbd286da861.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/8d6006e68126b2bd.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/9116dc99a2ecab1b.png)

