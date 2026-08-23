---
title: OpenClaw 的 AGENTS.md：写给 AI 的工作空间使用手册
feedId: 34419
source: 综合讨论
publishedAt: 2026-08-24
---

## 背景

最近在 OpenClaw 里接 MCP 工具链和自动化脚本，最明显的问题不是“能力不够”，而是会话间行为漂移：同一个仓库，换一个会话就可能搞错启动入口、把测试命令当成生产命令、或把产物写进源目录。后来我把这些约束收敛到工作空间根目录的 AGENTS.md，OpenClaw 会话启动时作为工作空间说明书加载，效果稳定很多。

## 问题

AGENTS.md 不是通用提示词，而是工作空间接口文档。它告诉 OpenClaw：这是什么项目、有哪些目录、用哪个命令跑测试、哪些目录不能动、MCP 工具怎么用、失败后先看哪里。没有它，每次新会话都要重新交代；有了它但如果写得模糊或过期，AI 一样会乱试。

## 做法/步骤

### 1. 定位文件

在 OpenClaw 工作空间根目录创建 AGENTS.md，注意大小写。若你的 OpenClaw 版本支持自定义加载路径，在配置中显式指定；否则放在默认扫描路径。实际使用中，建议先确认你的版本是否会读取根目录文件，避免写完不生效。

### 2. 固定骨架

我用七段结构：项目概览、目录结构、常用命令、环境与配置、约束与禁区、MCP/插件说明、验证方式。每段不必长，但信息要具体。

### 3. 写执行细节

命令段写清楚用途：

```markdown
## 常用命令
- 运行测试：`pytest -q --maxfail=1`
- 启动服务：`python src/main.py --port 8000`
- 构建前端：`cd web && npm run build`
```

约束段要写原因和替代路径：

```markdown
## 约束与禁区
- 不要直接修改 `data/raw/` 下文件；如需清洗，先复制到 `/tmp/openclaw-staging/`。
- 不要执行 `rm -rf` 或 `git push --force`；删除操作先列清单再确认。
- 生成产物统一放 `output/`，不要散落在根目录。
```

MCP 工具段给最小调用示例，而不是只写工具名。比如某个 MCP 工具需要特定 schema，就写一个最小 JSON 调用，让 OpenClaw 不必猜测参数结构。

### 4. 验证加载

重启 OpenClaw 会话，直接问：这个项目如何跑测试？禁止写哪些目录？如果回答偏离，检查加载路径或文件语法。建议每次改完 AGENTS.md 都做一次新会话验证。

### 5. 纳入版本管理

AGENTS.md 放仓库里，变更走 PR。避免本地临时改完不提交，导致远程会话读旧版。把它当接口文档维护，而不是随手笔记。

## 踩坑点

- **太长**：超过 200 行后，AI 对关键约束的注意力会下降。我一般控制在 120-180 行，把高频信息放前面。
- **只写禁止，不给替代**：否定约束容易被忽略。例如“不要碰 dist/”不如“dist/ 由 build 生成，不要手工修改；改 src/ 后运行 npm run build”。
- **过期**：命令已改但文档没更新，AI 会按旧命令执行。建议在 CI 里加简单检查，例如扫描 AGENTS.md 是否包含密钥、是否超过行数阈值。
- **模糊词**：“适当重试”“合理命名”对 AI 没用。写“重试 3 次，间隔 2 秒”或“文件名用 YYYYMMDD_HHMM.log”。
- **把敏感信息写入**：AGENTS.md 可能被自动附加、导出或分享，绝不能包含 token、密码、内网地址。
- **与 README 重复**：README 给人看，讲“为什么”；AGENTS.md 给 AI 看，讲“怎么做、边界在哪”。不要把两者写成一回事。

## 可复用建议

- **分层引用**：主 AGENTS.md 保持简洁，细节放 `docs/ai/commands.md`、`docs/ai/mcp.md`，通过文件工具按需读取。OpenClaw 不会默认读所有子文档，但主文件里写清“详细命令见 docs/ai/commands.md”。
- **模板化**：为 python-service、nextjs-app、data-pipeline 分别准备模板，新项目复制后改关键路径。
- **定期修剪**：每次调整目录、脚本或 MCP 工具后，同步删掉失效命令和路径。
- **用具体失败提示**：例如“如果 `pytest` 失败，先看 `output/test_report/` 下最新报告，不要直接改测试”。
- **本地验证**：写完后用新会话问同一组问题，确认 AI 的回答稳定且不越界。

## 总结

AGENTS.md 的目标不是让 AI 变聪明，而是把工作空间约定变成可执行、可验证的接口文档。它减少每次会话的解释成本，也降低 AI 在不确定时乱试的概率。工程化地写、版本化地维护、定期验证，比写一堆漂亮提示词更实际。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/b3d2faa04bc79df1.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/82f1097efb23dbfa.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/35de98de206abfae.png)

