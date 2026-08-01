---
title: OpenClaw 的 AGENTS.md：写给 AI 代理的工作空间使用手册
feedId: 31181
source: 综合讨论
publishedAt: 2026-08-01
---

## 背景：当 AI 代理开始在你的项目里“自由发挥”

OpenClaw 能够让 AI 代理在本地工作空间中直接执行 Shell 命令、读写文件、调用 MCP 工具。这在自动化脚本执行、项目搭建与代码重构场景里效率极高，但也立刻暴露出一个工程问题：**代理对项目约定、安全边界和操作习惯一无所知**。

在没有规则约束的情况下，一个看似无害的指令可能让代理把 `node_modules` 也纳入重构范围，或者顺手执行 `rm -rf` 清理“临时文件”。如果在团队中使用，不同成员对代理的预期不一样，混乱会更加严重。

正是为了解决这类问题，OpenClaw 设计了 `AGENTS.md` —— 一份放在项目根目录的声明式文档，专门用来告诉 AI 代理：**这间工作室里什么能做、什么不能做，以及怎么做才符合工程标准**。它不是可选建议，而是让代理稳定工作的基础设施。

## 问题：没有规范时，代理会踩哪些坑

- **安全失控**：代理可能执行危险的系统命令或修改 `.git` 目录。
- **破坏工程约定**：代理不了解项目的 lint、test、commit 规范，产出的代码需要大量人工修正。
- **MCP 工具滥用**：比如在不该操作数据库的自动化流程里调用了 `sqlite` 或 `github` 服务。
- **敏感文件泄露**：代理可能读取 `.env` 或密钥文件并错误地写入日志或输出。

这些问题不是“调 prompt”能根治的，因为提示词无法感知本地文件系统结构和团队协作规则。`AGENTS.md` 的出现，就是把看不见的上下文变成一份机器可读的章程。

## 做法：从零开始编写一份工程化的 AGENTS.md

### 1. 确定文件位置与解析优先级
OpenClaw 会按以下顺序查找并合并规则（后加载的覆盖先前的）：
- 全局配置：`~/.openclaw/AGENTS.md`
- 项目级配置：`<项目根>/AGENTS.md`
- 通过环境变量 `OPENCLAW_AGENTS_PATH` 指定的额外文件

**工程建议**：项目级文件提交到版本库，保证团队一致；个人偏好放在全局配置中。

### 2. 定义工作区边界
```markdown
# workspace
path: .
readonly:
  - .git
  - node_modules
  - .env
  - secrets/
ignore:
  - "*.log"
  - dist/
```
- `readonly`：代理只能读取，不可修改或删除。
- `ignore`：连读都不需要，提升遍历效率。

### 3. 声明可执行命令白名单
```markdown
# commands
allow:
  - npm run *
  - npx vitest *
  - ls
  - cat
  - git status
  - git diff
deny:
  - rm -rf
  - sudo
  - shutdown
  - curl
```
注意：`allow` 支持通配符，但建议尽量缩小匹配范围。`deny` 拥有最高优先级，即使被 `allow` 覆盖也会否决。

### 4. 启用或限制 MCP 工具
```markdown
# mcp
servers:
  - filesystem
  - github
  - sequential-thinking
disabled:
  - puppeteer
```
只有被显式声明的 MCP 服务才会在代理工作流中可用。如果你不写 `github`，代理就无法操作 Issue 或 PR——这比任何提示词都可靠。

### 5. 写入项目级任务指引
这一节直接嵌入到 `AGENTS.md` 底部，用自然语言描述规则：
```markdown
# instructions
- 代码风格遵循项目下的 .eslintrc.js 和 .prettierrc。
- 任何修改后必须运行 `npm run test -- --changed`。
- 提交消息必须符合 conventional commits 格式。
- 不要主动修改 package-lock.json，除非明确要求。
```
OpenClaw 将在每次任务启动时，把 `instructions` 注入到代理的系统上下文，相当于给代理发了一份“员工手册”。

### 6. 验证规则是否生效
创建一个简单测试：要求代理输出当前工作区只读目录列表。如果它正确识别了 `readonly` 字段，并在后续操作中拒绝删除相关文件，说明规则生效。

推荐在 CI 里加入一个轻量探针：调用 OpenClaw 执行一条会触发 `deny` 的命令，预期返回权限拒绝错误。

## 踩坑点

- **路径区分大小写与跨平台**：在 macOS 上 `Readonly: .Git` 可能被放过，需统一为小写。
- **命令白名单写 `*` 会放行一切**：曾经有个项目写了 `allow: *`，结果代理执行了 `git push --force`。绝对不要把 `*` 单独用在 `allow` 里。
- **嵌套项目未处理**：monorepo 场景下，子包也需要自己的 `AGENTS.md`，否则代理可能套用根规则导致过严或过松。
- **MCP 服务名不匹配**：`disabled` 中的名字必须和 MCP 配置里 `name` 字段完全一致，多一个空格都会失效。
- **指令和解析器冲突**：避免在 `instructions` 中使用 YAML 风格的 `# comment`，会被当成注释忽略，使用 Markdown 段落。

## 可复用建议

1. **维护团队模板**：创建一个 `AGENTS.template.md`，新项目直接复制并调整。
2. **分层管理**：将通用规则放在全局配置，项目特定规则放在项目文件中，避免大量重复拷贝。
3. **定期审计**：随着项目依赖和工具链变化，每季度检查一次 `allow` 命令列表是否仍然必要且安全。
4. **与环境变量联动**：可以通过 `${ALLOWED_TOOLS}` 这类占位符在启动时动态注入权限，适用于多环境部署。
5. **存储为代码**：将 `AGENTS.md` 纳入版本控制，通过 PR 评审变更，就像管理 `.github/workflows` 一样。

## 总结

`AGENTS.md` 把对 AI 代理的信任从模糊的口头约束变成了清晰、可版本化、可审计的工程配置。它不消除风险，但把风险的敞口压缩到了你能看见的地方。如果你已经在用 OpenClaw 做本地自动化，那今天就应该为你的项目补上这份工作空间使用手册——花 15 分钟写完，后面每一次自动化执行都会替你省下不止 15 分钟的清理时间。

---

