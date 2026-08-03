---
title: OpenClaw 的 AGENTS.md：给 AI 行动一个可靠的“工作手册”
feedId: 31453
source: 综合讨论
publishedAt: 2026-08-03
---

# OpenClaw 的 AGENTS.md：给 AI 行动一个可靠的“工作手册”

## 背景：当 Agent 不再“听话”

OpenClaw 这类以工作空间为核心的 Agent 框架，打通了本地文件、Shell、MCP 工具与外部 API。你给它一个自然语言目标，它就能在目录里查文件、改配置、跑测试，甚至调用第三方插件。但问题随之而来：没有明确的行为边界，Agent 很容易过度自由——删错文件、循环调用工具、访问不该碰的目录，或者在错误路径上钻牛角尖。

传统的 `system prompt` 只能承载有限的通用规则，一旦涉及特定项目的目录结构、自定义 MCP 服务、权限范围，就很难说清楚。于是 OpenClaw 引入了 **AGENTS.md**：一份放在项目根目录的声明式 Markdown 文件，作为 Agent 启动时加载的“工作空间使用手册”。

## 问题：缺乏上下文约束带来的三座大山

1. **环境盲区**：Agent 不知道 monorepo 里哪个是主服务目录、哪个是废弃实验，乱翻一通浪费 Token。
2. **工具滥用**：MCP 工具暴露了数据库写权限，Agent 在调试时直接执行了 `DROP TABLE`。
3. **流程发散**：用户只说“修 bug”，Agent 可能跳过了 lint→test→commit 的约定流程，导致低质量提交。

AGENTS.md 的出现，就是把这些隐性知识写成显式的、可被 AI 解析的约束。

## 做法与步骤

### 1. 文件位置与加载机制
OpenClaw 启动工作空间时会扫描 `AGENTS.md`，如果存在，会将其内容作为系统级上下文注入，优先级高于默认 prompt。可以在 `openclaw.yaml` 中指定自定义路径，但默认约定是根目录的 `AGENTS.md`。

### 2. 编写指南：一个标准 AGENTS.md 长什么样

推荐结构如下：

```markdown
---
workspace: my-project
role: backend-dev
mcp_servers: [filesystem, github, slack]
safety:
  readonly_dirs: [/etc, ~/.ssh]
  allow_delete: false
---

# 工作空间概览
- `src/` 主服务代码，Node.js + Express
- `tests/` 使用 vitest
- `scripts/` 运维脚本，非必要勿动

# 可用工具与调用约定
- `filesystem`: 可读写 ./data 和 ./logs，其余只读
- `github`: 仅允许创建分支、PR，禁止直接 push main
- `slack`: 仅允许向 #alerts 频道发送消息

# 任务执行流程
1. 理解需求，若模糊则提问澄清
2. 在 `src/` 中定位相关模块
3. 修改代码，遵循项目 ESLint 规则
4. 运行 `npm test` 和 `npm run lint`
5. 若通过，使用 git 工具新建分支并 commit
6. 向用户汇报变更摘要

# 错误处理
- 若 3 次重试后仍失败，暂停并请求人工介入
- 禁止绕过测试直接提交
```

### 3. 与 MCP 工具的联动
MCP 工具的名称、参数往往在 `openclaw.yaml` 里定义，但 AGENTS.md 可以进一步限定其**使用场景**和**参数模版**。例如，上面的 `github` 工具被限制只能创建分支而不能直接推送 main，Agent 在解析时会参考这些规则，减少越权操作。

### 4. 多环境与变量
团队协作时，`AGENTS.md` 中的路径或域名常与环境相关。建议使用占位符配合环境变量：`${DATA_DIR}`、`${SLACK_CHANNEL}`，OpenClaw 会在加载时替换。这样同一份手册可以适配开发、CI、预发布环境。

## 踩坑实录

- **相对路径歧义**：Agent 解析 `./data` 时可能以自身进程的 CWD 为基准，而非工作空间根目录。解决：AGENTS.md 中明确所有路径相对于工作空间根，并在根目录放一个 `.openclaw-root` 标记文件。
- **工具名大小写不一致**：MCP 服务器注册时习惯小写连字符（`file-system`），但 AGENTS.md 中写成驼峰，导致规则不匹配。务必与 `openclaw.yaml` 中的 `name` 字段严格一致。
- **权限过宽**：很多人把 `allow_delete: true` 设成全局，结果 Agent 在一次“清理无用文件”的任务中删除了 `.env`。建议删除操作默认关闭，必须明确列出可删除的目录白名单。
- **更新不生效**：修改 `AGENTS.md` 后，正在运行的 Agent 会话不会自动重载。要么重启工作空间，要么在任务里主动触发 `/workspace reload`（如果支持）。自动化流水线中，建议在变更后立即重启 agent 容器。
- **多 Agent 冲突**：如果一个工作空间被多个不同角色的 Agent 共同访问（如 “builder” 和 “reviewer”），单份 AGENTS.md 可能顾此失彼。此时可创建 `AGENTS.builder.md`、`AGENTS.reviewer.md`，并在 OpenClaw 配置中通过 `agent_rule_file` 按 Agent ID 分发。

## 可复用建议

1. **模板化**：提炼团队通用的 AGENTS.md 骨架，放入脚手架中。新项目 `openclaw init` 时自动生成。
2. **引入版本控制**：AGENTS.md 应纳入 Git 管理，与代码同源。任何人修改工作流，都必须同步更新它。
3. **语义分段**：使用 `## 权限边界`、`## 任务协议` 等固定标题，便于 Agent 按章节检索，也方便将来用脚本做合规检查。
4. **与 CI 结合**：在 PR 时通过 lint 工具检查 AGENTS.md 中的 MCP 工具名是否有效、路径是否存在，防止规则失效。
5. **保留“安全词”**：设置一个始终监听的停止指令，如“紧急停止”，写入 AGENTS.md 的安全区。Agent 在执行长任务前会自检该指令，以备人工打断。

## 总结

AGENTS.md 不是什么银弹，它只是把人类对 Agent 的信任，从模糊的 prompt 变成了可审计、可版本化、可测试的规则文件。在 OpenClaw 生态里，它像一面镜子，照出我们对自动化流程的真实理解程度。写得越清晰，Agent 的自主行为就越可控；写得越草率，就越容易收获一场由 AI 主导的“行为艺术”。工程实践中，值得花一个下午认真写一份 AGENTS.md，它会成为你工作空间里最划算的投入。

---

