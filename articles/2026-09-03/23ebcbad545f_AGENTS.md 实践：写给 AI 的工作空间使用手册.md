---
title: AGENTS.md 实践：写给 AI 的工作空间使用手册
feedId: 35920
source: 综合讨论
publishedAt: 2026-09-03
---

## 背景

AGENTS.md 这个约定近一年在 agent 工具链里逐渐成型：在工作区根目录放一个 Markdown 文件，agent 启动时会自动读取，作为对这个环境的"先验知识"。OpenClaw 遵循同样的约定——agent 每次进入工作区，第一件事就是找它。可以简单理解为：README.md 写给人看，AGENTS.md 写给 AI 看。

## 问题

没有这份手册时，agent 每个会话都在盲猜：

- 构建用 `npm run build` 还是 `make`？跑测试前要不要先起数据库？
- `dist/`、`*.gen.ts` 是生成物，agent 却当成普通代码去改；
- 团队约定提交信息走 Conventional Commits，agent 每次自由发挥。

这些知识其实都存在——在老成员脑子里、在 wiki 里、在某个 ISSUE 里，但 agent 够不着。塞进系统提示词可以救急，但不可版本化、不跟随仓库走，换个 agent 就失效。

## 做法

我们工作区的 AGENTS.md 大约 40 行，骨架如下：

```markdown
# 项目概览
一句话说清这是什么、技术栈、入口在哪。

# 常用命令
- 安装 / 构建 / 测试 / lint 的准确命令
- 测试的前置条件

# 目录说明
- src/ 手写代码
- generated/ 自动生成，禁止手改
- scripts/ 一次性脚本，可参考勿复用

# 红线
- 不要修改 migrations/ 下的历史文件
- 不要直接 push 到 main
- 依赖变更必须说明理由

# 协作偏好
- 提交信息用 Conventional Commits
- 回复与注释用中文
```

几条写法原则：

1. **短**。agent 每轮都带着这份文件，40 行以内信息密度最高，超过一屏，重要规则就被稀释。
2. **具体可执行**。"写高质量代码"没用，"测试文件命名为 `*.spec.ts` 放在 `__tests__/`"才有用。
3. **分层**。Monorepo 可在子目录再放一份 AGENTS.md，只写该子包特有约定，根文件保持全局视角。
4. **迭代**。agent 同一件事犯错两次以上，就值得写成一条规则。

## 踩坑点

- **写成项目文档**。把架构演进史、设计取舍全塞进去，agent 读得累，重点反而丢了。历史背景放 docs/，AGENTS.md 只放"操作指令"。
- **多份规则文件漂移**。仓库里同时有 CLAUDE.md、.cursorrules，各改各的，agent 读到矛盾指令。建议选一个作为唯一事实源，其余用符号链接或构建时生成。
- **写死易变信息**。端口号、当前迭代需求、密钥路径都别放，这些该进配置或环境变量。
- **只写不改**。约定变了手册没变，agent 会拿着过期地图认真迷路。把它当代码对待：进版本库、走 review。

## 可复用建议

- **冷启动技巧**：让 agent 先自己浏览工作区，输出一份"它理解的约定清单"，你在其基础上删改。比从零写快得多，还能暴露 agent 的错误假设。
- **维护"踩坑记录"小节**：只收录真实发生过的事故，每条一行。这一节往往是最值钱的部分。
- **半自动校对**：定期让 agent 复述 AGENTS.md 并指出与代码现状不符的条目，人工确认后更新。

## 总结

AGENTS.md 的本质，是把团队里口口相传的工作区知识，固化成一份 agent 可稳定消费的产物。成本约一小时，收益是每个会话少几十次纠偏。如果你的 OpenClaw 工作区还没有这份文件，建议今天就用上面的骨架补一份——先短后长，先命令后约定，让它跟着仓库一起演进。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/311c291b83fe5881.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/baff637269410526.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/904d33ae8c748c87.png)

