---
title: OpenClaw 的 AGENTS.md：为 AI Agent 编写一份工作空间使用手册
feedId: 32023
source: 综合讨论
publishedAt: 2026-08-07
---

# OpenClaw 的 AGENTS.md：为 AI Agent 编写一份工作空间使用手册

## 背景：当 Agent 走进你的项目

OpenClaw 让 AI Agent 可以直接在工作空间中执行命令、读写文件、调用 MCP 工具链。但 Agent 默认对项目几乎一无所知 —— 不知道你的构建命令是 `pnpm build` 还是 `make`，不知道测试数据库容器的启动脚本藏在 `scripts/dev/` 里，更不知道团队禁止直接修改 `generated/` 目录。

这类信息通常散落在 `README`、`CONTRIBUTING` 或是老员工的即时消息里。Agent 没有这些背景，就会出现“能跑但错了”的行为：用错包管理器、格式化代码时覆盖手动对齐、在错误的路径下生成文件。更隐蔽的问题是，同一个任务让 Agent 执行 3 次，可能得到 3 种不同风格的输出——因为没有一份明确的规范来约束它的行为边界。

AGENTS.md 就是为这个问题设计的。它是一份放在项目根目录（或任意子目录）下的 Markdown 文件，是专门写给 AI Agent 的项目工作空间说明。它不是给人看的 README 的翻版，而是一组机器可读、行为导向的指令集，用来降低 Agent 在项目中“试错—纠正—再试错”的循环成本。

## 核心问题：Agent 的行为需要一个锚点

没有 AGENTS.md 时，OpenClaw 的 Agent 默认依赖系统 prompt 和 MCP 工具的上下文。但这两种来源都不可靠：

- 系统 prompt 是全局的，无法反映单个仓库的特定约束。
- MCP 工具的动态上下文（例如打开的文件、终端输出）虽然实时，但缺少静态规则层，Agent 容易在“摸索”中踩坑。

一个典型场景：你让 Agent “加一个健康检查接口”。Agent 发现项目用 Go，决定用标准库。但实际上项目在 `internal/transport/http` 下有统一的路由注册方式，还有自定义的 `httputil` 响应封装。Agent 不知道这些，写出的代码能编译，但破坏了分层结构，需要人工大量修正。如果 AGENTS.md 明确写了“所有 HTTP handler 必须使用 `httputil.Wrap` 注册，并放在 `internal/transport/http/handler` 下”，Agent 就会先校验规则再生成代码，而不是先犯错后修复。

## 做法：编写一份机器友好的 AGENTS.md

AGENTS.md 不是一次性写完的文档，应该随着项目迭代持续修剪。一份可用的 AGENTS.md 通常包含以下区块：

### 1. 项目速览（Agent 必须知道的硬事实）
```markdown
# Project: openclaw-engine
- Language: Go 1.22
- Module: github.com/openclaw/engine
- Build: `go build ./...`
- Lint: `golangci-lint run`
- Test: `go test -race -count=1 ./...`
```
这里只放 Agent 执行任务时 100% 会用到的命令，不放安装指南（人在首次 clone 时会阅读，但 Agent 不需要）。

### 2. 目录与职责说明
用简短的一句话说明关键目录的用途和约束，帮助 Agent 决定文件该放在哪：
```markdown
## Directory Map
- `cmd/` – 入口程序，不允许放业务逻辑。
- `internal/domain/` – 纯领域模型，禁止导入任何基础设施包。
- `internal/adapters/` – 外部系统适配实现，一个文件只对应一种数据库操作。
- `generated/` – 由 `buf generate` 自动生成，**严禁手动编辑**。
- `scripts/dev/` – 本地开发用脚本，不要在生产路径中引用。
```

### 3. 规则与约束（可执行的行为边界）
规则要具体到“怎么检查是否违规”。模糊的“保持代码整洁”没有用，Agent 需要类似这样的约束：
```markdown
## Rules
1. 所有公开函数必须已有或新增对应表驱动测试。
2. 错误消息以英文小写开头，不带结尾标点。
3. 新引入的依赖必须使用 `go get` 而非手动编辑 go.mod。
4. 对 `generated/` 的任何直接修改都视为错误；如需改协议请运行 `buf generate`。
```

### 4. 常用任务模板（减少 Agent 猜测）
把高频任务拆解成可执行的步骤，Agent 可以按编号依次执行：
```markdown
## 常见任务
**新增一个 RPC 方法：**
- [ ] 在 `proto/service/v1/` 下定义请求/响应消息与服务。
- [ ] 运行 `buf generate` 生成桩代码。
- [ ] 在 `internal/transport/grpc/handler_v1.go` 中实现处理函数。
- [ ] 在 `internal/domain/` 下提供对应的用例方法。
- [ ] 添加测试并确认 `go test ./internal/transport/grpc/...` 通过。
```

## 踩坑点：AGENTS.md 不是银弹

在实践中，有几个常见问题值得注意：

- **过度冗长导致 Agent 忽略**：一份 200 行的 AGENTS.md 对 Agent 来说和不存在区别不大。Agent 受上下文窗口限制，只有前 800 字左右的指令最容易被遵守。因此必须用最精炼的表达，规则尽量用条目式，每一条都可以被快速检验。
- **动态信息不要写进 AGENTS.md**：端口号、数据库连接、临时凭证等变量信息容易过期，Agent 读到错误的端口可能直接导致任务失败。这类信息应该通过 MCP 工具（如环境变量读取、配置服务）动态获取，AGENTS.md 只需说明从哪里获取，例如：“运行期配置通过 `scripts/dev/env.sh` 注入，不要硬编码。”
- **与系统 prompt 的优先级问题**：如果 AGENTS.md 说“使用 `pnpm`”，但 Agent 的系统 prompt 默认倾向 `npm`，最终行为取决于 OpenClaw 内部的指令融合策略。目前更可靠的方案是在 AGENTS.md 中明确声明“以下规则覆盖全局默认行为”，并尽量将关键命令前置。
- **子目录覆盖**：当项目包含多语言或多运行时，仅一个根 AGENTS.md 不够。可以在子目录下放置独立的 AGENTS.md，OpenClaw 会合并上下文。但要注意，不要在不同文件中定义相互矛盾的规则，Agent 不会做仲裁。
- **Agent 的“创造性”规避**：有时 Agent 会找到一个技术层面上不违反规则但违背意图的路径。避免这种情况需要加入负面例子，比如“不要为了绕过 lint 而使用 `//nolint` 注释，除非附带说明原因”。

## 可复用建议：让 AGENTS.md 保持生命力

1. **随 PR 一起更新规则**：当某条规则被破坏且人工修复后，立刻更新 AGENTS.md 使之可检查。这是唯一切实的长期维护策略。
2. **用注释标记“仅 Agent 可见”指令**：如果某段说明对人类阅读意义不大，但对 Agent 很关键（例如“检查 `buf.gen.yaml` 的最新版本号再生成”），可以用 `<!-- AGENT: 指令内容 -->` 的注释形式嵌入，Agent 能读到，但不会干扰人类阅读。
3. **与 MCP 工具联动**：你可以写一个 MCP 插件，读取 AGENTS.md 中定义的规则并在 Agent 执行前注入提示。更进一步，可以在任务完成后运行验证脚本，如果检测到规则违例，将失败信息作为新的上下文反馈给 Agent，形成一个封闭的校正回路。
4. **分层架构的 AGENTS.md 模板**：对于单体仓库，可以在不同模块放置精简版，只声明模块特有的约束；根文件保留跨模块通用约束。避免把所有信息堆在一起。
5. **测试 AGENTS.md 的有效性**：一个简单的方法是，故意让 Agent 执行一个曾出过错的旧任务，观察它是否因 AGENTS.md 的存在而直接走上正确路径。如果还会犯同样错误，说明规则不够精确。

## 总结

AGENTS.md 是给 AI Agent 的“工作间使用手册”，本质上是用静态结构化信息降低 Agent 在动态环境中的决策成本。它不能代替代码审查、测试或团队约定，但能大幅减少低级错误和风格不一致的问题。

写一份能用的 AGENTS.md，不需要覆盖所有细节，只需要把那些“Agent 做 10 次会错 8 次”的东西用可检查的规则固定下来。把它当作项目 CI 的延伸——像 linter 规则一样持续演进而非一蹴而就。当 Agent 的行为变得稳定、可预测，你才真正从和 AI 的“纠错博弈”中抽身出来，开始利用它完成更大的工程任务。

---

