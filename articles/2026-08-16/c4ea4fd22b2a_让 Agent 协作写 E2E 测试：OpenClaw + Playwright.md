---
title: 让 Agent 协作写 E2E 测试：OpenClaw + Playwright MCP 的工程化实践
feedId: 33461
source: 综合讨论
publishedAt: 2026-08-16
---

## 背景

E2E 测试的价值不用多说，但维护成本一直不低。页面一改，选择器失效；登录态、权限、多角色流程反复配置；断言写得随意，跑挂了又没人愿意修。

Agent 协作出现后，团队开始尝试让 AI 直接生成测试用例。我们也在 OpenClaw-CN 的实践里试了一段时间，结论是：直接丢一句“帮我写登录页 E2E 测试”基本不可用。真正能落地的做法，是把探索、生成、执行、修复拆成可约束的小步骤，并通过 MCP 工具把项目规范带进 agent 上下文。

## 问题

我们在早期尝试中遇到几个具体问题：

1. AI 对页面结构的假设与真实 DOM 不一致，经常使用不存在的 class。
2. 登录态和多角色流程无法自动处理，AI 会写成每个用例都重新登录。
3. 生成的代码能跑，但断言很弱，通常只验证“页面打开了”。
4. 每个项目的选择器约定不同，AI 默认喜欢 `text=` 或 `nth=`，导致用例脆弱。

这些问题不是模型能力不够，而是缺少工程约束和反馈闭环。

## 做法 / 步骤

我们使用 OpenClaw 编排三个角色 agent：

- **Explorer**：用 Playwright MCP 浏览页面，产出结构化页面描述。
- **Test-Writer**：根据页面描述和业务约定生成 Playwright spec。
- **Runner**：执行测试，收集失败信息并回传给 Test-Writer 修复。

具体步骤如下：

### 1. 接入 Playwright MCP

在 OpenClaw 中注册 MCP server，暴露浏览器工具：

```yaml
mcp_servers:
  playwright:
    command: npx
    args: ["@playwright/mcp@latest"]
```

我们主要使用 `browser_navigate`、`browser_snapshot`、`browser_click` 和 `browser_take_screenshot`。注意是让 agent 读取可访问性树快照，而不是直接读 HTML，这样能减少大量噪音。

### 2. 定义业务上下文文件

把关键路由、登录账号、角色、选择器策略、禁止使用的选择器写成一个 markdown 文件，作为每个 agent 的上下文。例如：

- 优先使用 `data-testid`
- 其次使用 `role` + accessible name
- 禁止使用 `nth-child`、裸 CSS class
- 禁止对时间戳、随机值、表格精确行数做断言

这个文件并不复杂，但能显著减少返工。

### 3. 探索页面

Explorer agent 按 route 列表逐个访问页面，用 `browser_snapshot` 获取结构，输出 page-object 描述。描述内容包括：关键操作、表单项、触发条件、预期结果。人工确认后再进入生成环节，避免一步到位返工。

### 4. 生成用例

Test-Writer 只使用 page-object 描述和业务约定生成 spec。要求每个用例包含 arrange / act / assert 三部分，断言只针对稳定业务状态，比如“状态标签变为已提交”，而不是“表格出现 5 行数据”。

### 5. 执行与修复

Runner agent 在 CI 或本地执行 spec，收集失败截图、DOM 快照、console 错误，回传给 Test-Writer。最多修复 3 轮，超过 3 轮直接转人工，不在 agent 里无限重试。

### 6. 人工评审

生成后的代码进入 PR。人工重点检查业务断言是否合理、是否重复登录、是否覆盖了权限分支。Agent 负责生成和初步修复，人负责判断业务正确性。

## 踩坑点

- **选择器脆弱**：我们踩过登录按钮文案从“登录”改成“Log in”，`text=` 选择器直接挂掉。必须从上下文里强制优先 `data-testid` 和 `role`。
- **登录态复用**：Agent 默认每个用例都登录，导致运行慢、产生脏数据。用 Playwright `storageState` 在 `beforeAll` 保存登录态，并明确告诉 Test-Writer 不要在用例内重复登录。
- **动态数据断言**：时间戳、随机数、表格行数不能做精确断言。需要给规则：只断言稳定部分，如行存在、状态标签变化。
- **Explorer 过度探索**：agent 会点进未知弹窗、跳到外部站点。必须限制 host allowlist 和最大操作步数。
- **上下文爆炸**：`browser_snapshot` 返回大 DOM，token 消耗很快。可以先做 route 级快照，过滤 header/footer，或封装项目专用 MCP 工具返回精简结构。
- **修复循环失控**：失败修复超过 3 轮就人工处理，否则 agent 会反复在同一个地方打转。

## 可复用建议

- 把业务约定固化成一个“测试上下文文件”，跟随仓库版本管理。
- 封装项目专用 MCP 工具，比如 `get_page_structure(route)` 返回去噪后的可访问性树，`login_as(role)` 返回 `storageState` 路径。这比直接用裸 Playwright MCP 更省 token，也更容易约束 agent。
- 先让 agent 生成烟雾流程和关键业务路径，稳定后再扩展覆盖面。
- 在 CI 中增加“生成—运行—报告”的 job，不让 agent 在开发机里随意跑浏览器。
- 保留人工评审，只把重复性页面覆盖和失败修复交给 agent。

## 总结

Agent 协作写 E2E 测试，真正可行的不是“自动生成完美用例”，而是把页面探索、用例编写、失败修复拆成可约束的小步骤，配合 MCP 工具把项目规范带进上下文。这样产出的是可评审、可维护的测试，而不是一次性脚本。目前我们仍然保留人工评审，但重复性覆盖和失败修复的负担已经明显下降。

---

