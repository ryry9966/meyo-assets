---
title: OpenClaw 的 AGENTS.md：写给 AI 的工作空间使用手册
feedId: 35265
source: 综合讨论
publishedAt: 2026-08-29
---

## 背景

OpenClaw 的 agent 在做自动化任务时，经常不是能力不够，而是上下文不一致。同一个工作区，人知道 `src/` 是主服务、`data/` 不能动、测试要带 `--runInBand`，但 agent 第一次进入时只能靠目录名猜，或者在长会话里慢慢忘记。AGENTS.md 的作用，就是把这些“工作空间使用约定”固化下来，作为 AI 进入工作区后的第一份手册。

## 问题

没有 AGENTS.md，或者写得太随意，通常会遇到三类故障：

1. **启动成本高**：agent 反复探测目录、试错命令，消耗 token，还容易把旧脚本当入口。
2. **规则漂移**：把关键约束只放在 prompt 里，长任务中途容易丢；尤其多个 agent 或 MCP 工具协作时，各自理解不一样。
3. **安全边界模糊**：没有写明禁止操作，agent 可能自动 push、删除临时目录、把密钥写进日志。

## 做法/步骤

1. **放置与加载**：在项目根或 OpenClaw 工作区根放 `AGENTS.md`。如果启动器支持自动加载，确认路径；不支持就在 system prompt 或项目规则里显式要求 agent 先完整读取。
2. **内容结构**：按可执行信息组织，不要写愿景。核心模块包括：目录职责、常用命令、环境变量、禁止操作、MCP/插件约定、验证方式。
3. **维护最小模板**：

```markdown
# AGENTS.md
执行任何任务前，先完整读取本文件，并复述关键约束。

## 目录职责
- src/：主服务，禁止在 src/ 外新建业务模块
- scripts/：一次性脚本，不要放生产代码
- data/：数据目录，只读，禁止修改

## 常用命令
- 安装：pnpm install
- 测试：pnpm test -- --runInBand
- 构建：pnpm build
- 启动：pnpm dev --port 3000

## 环境变量
- 必填：DATABASE_URL、REDIS_URL
- 禁止把任何环境变量值写入日志或输出

## 禁止操作
- 禁止自动 git push
- 禁止删除 data/、logs/
- 禁止运行 rm -rf 或等价命令
- 修改配置前先备份

## 验证
完成后必须运行：pnpm test -- --runInBand && pnpm typecheck
```

4. **迭代回写**：每次 agent 踩坑后，把人工纠正写回 AGENTS.md，避免下次重复。

## 踩坑点

- **写成 README**：给人看的背景太多，agent 需要的是路径、命令、禁止项，不是项目愿景。
- **绝对路径/硬编码**：换容器、换机器就失效，优先用相对路径或环境变量。
- **安全边界不写**：不要假设 agent 会“自觉”。禁止自动 push、删除、读取敏感文件要明确列出。
- **文件过长**：超过 200 行后，模型容易忽略尾部；复杂内容拆到 `docs/` 并链接。
- **只写不更新**：过期规则比没有更危险，agent 会按错误约定执行。
- **多层级作用域混乱**：明确根 `AGENTS.md` 优先，子目录只放增量规则。

## 可复用建议

- 团队统一模板，新项目复制后填空，而不是每次从零起草。
- 顶部固定一句“先读本文件并复述关键约束”，能显著降低起始漂移。
- 把验证命令写进去，让 agent 完成后自检，而不是只交代码。
- 用 git 管理 `AGENTS.md`，变更走 PR，避免个人随口改规则。
- 配合 MCP filesystem 限制可写目录，并在 `AGENTS.md` 里同步声明。
- 多 agent 协作时，写清楚各自可写目录和共享只读目录。

## 总结

AGENTS.md 不是另一份给人看的文档，而是 AI 的工作空间约束。它解决的是上下文一致性和安全边界，不是模型能力问题。好的 AGENTS.md 短、具体、可执行、常更新；差的长篇、模糊、过期。先把根目录那 80 行写好，比堆更多插件更有效。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/49f06fa0829be51b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/be987f939da3c4a1.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/107ada9e96641723.png)

