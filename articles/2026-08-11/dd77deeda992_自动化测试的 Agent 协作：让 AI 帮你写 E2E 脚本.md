---
title: 自动化测试的 Agent 协作：让 AI 帮你写 E2E 脚本
feedId: 32602
source: 综合讨论
publishedAt: 2026-08-11
---

## 背景

端到端（E2E）测试是保障核心用户路径不被破坏的最后一道防线，但编写和维护成本极高。一个登录流程、一次支付操作，往往因为一个 UI 调整就让整条测试链断裂。即便团队已经有了 Playwright 或 Cypress 的脚本，定位元素、处理异步、维护断言仍然是投入产出比很低的重复劳动。

大模型在理解页面结构和生成操作步骤上表现出明显潜力。但在没有真正操作浏览器的能力时，AI 只能给出静态建议——你可以把 DOM 贴给它，但无法闭环验证。Agent 协作的思路正好填补了这个缺口：**给 Agent 挂载浏览器控制能力，让它自己看、自己做、自己修**。

本文以 OpenClaw 的 Agent 框架为例，结合 MCP（Model Context Protocol）的浏览器工具，搭建一个能探索页面、生成测试、执行并自我修正的 E2E 辅助流水线。不聊“颠覆”，只讲可复现的工程实践。

## 问题拆解

一条合格的 E2E 测试需要解决三个层面的问题：

1. **理解页面**：哪些元素可交互，交互顺序是什么，会产生什么副作用。  
2. **表达测试**：选对选择器，处理等待，写断言。  
3. **自我修复**：页面变化时，测试能否低损地跟上。

单次让 AI 输出代码很容易，但落地就难在第三步：生成的测试跑一次就失败，修起来比手写还累。Agent 循环的价值在于：**把失败信息喂回模型，重新调整步骤，再执行，直到通过或达到最大重试**。这不是魔法，而是一个带反馈执行环境。

## 具体做法

### 1. 启动浏览器 MCP Server

我们使用基于 Playwright 的 MCP 服务器，将浏览器操作暴露为标准化工具：

```bash
npx @anthropic/mcp-server-playwright
```

该服务提供 `browser_navigate`、`browser_snapshot`、`browser_click`、`browser_type`、`browser_evaluate` 等工具。启动后，Agent 可以通过 MCP 协议直接调起无头浏览器实例。关键在于 **snapshot 工具会根据可访问性树生成紧凑的页面结构**，而不是返回整个 DOM，这大幅减少了 token 消耗并提高了模型的定位准确度。

### 2. 设计 Agent 角色

在 OpenClaw 中创建一个专门的 agent，核心 system prompt 如下（简化版）：

```text
你是一个 E2E 测试工程师。你拥有一个浏览器工具集来完成工作。
流程：
1. 使用 browser_navigate 打开目标页面。
2. 使用 browser_snapshot 获取页面的可交互元素列表（每个元素带有 ref 编号）。
3. 依据常见的用户路径，生成一组自然语言测试场景。
4. 针对每个场景，使用 ref 编号执行点击、输入、断言等操作。每次操作后再次抓取 snapshot 以确认状态变化。
5. 将整个执行过程转换为可复用的 Playwright 脚本，尽量使用基于 ref 的语义选择器（如 getByRole、getByText），避开动态 class。
6. 如果脚本执行失败，分析错误原因，调整选择器或等待策略，重新执行直到通过或重试上限。
```

这样设计的优点是**分步执行、反馈闭环**，而不像单步生成那样期待一次完美。

### 3. 实际运行流水线

用一段 CLI 指令触发 Agent 任务：

```bash
openclaw run --agent e2e-writer --params '{"url": "https://example.com/login"}' --max-steps 20
```

Agent 会依次：

- 导航到登录页；
- 识别邮箱输入框、密码框、登录按钮的 ref 编号；
- 执行输入、点击登录；
- 快照验证是否跳转到仪表盘，并提取导航菜单文本来做断言；
- 生成如下代码（示意）：

```javascript
test('login with valid credentials', async ({ page }) => {
  await page.goto('https://example.com/login');
  await page.getByRole('textbox', { name: 'Email' }).fill('user@example.com');
  await page.getByRole('textbox', { name: 'Password' }).fill('correct-password');
  await page.getByRole('button', { name: 'Sign in' }).click();
  await expect(page.getByText('Dashboard')).toBeVisible();
});
```

如果某一步失败（例如密码框的 label 在 snapshot 中显示为“输入密码”），Agent 会捕获失败截图和错误信息，自动调整选择策略并重新尝试。

## 踩坑与经验

### 选择器脆弱性

直接拿 snapshot 中的 ref 转换为 CSS 或 XPath 仍然容易脆断。我们的经验是要求 Agent 优先输出 `getByRole`、`getByLabel` 这类语义选择器，同时保留 ref 作为兜底。可以额外加一条规则：**不允许使用动态生成的 class 或结构位置（如 nth-child）**。

### 认证状态保持

很多应用 E2E 需要登录态。每次 agent 执行都重新登录会浪费大量 token。可以在任务参数中传入 `storageState` 路径，让 Playwright 复用已有状态，Agent 只需关注测试本身。

### 网络异步与不稳定性

单靠 snapshot 无法覆盖所有异步场景（如接口返回后才渲染按钮）。建议 Agent 利用 `browser_wait_for` 工具等待文本或元素出现，而不是硬编码 sleep。错误的处理尤其重要：当 Agent 自己写的测试失败时，它的“自我修复”可能陷入死循环。需要设定重试上限和差异度比较，避免微小样式变化触发无意义的全面重写。

### 安全与隔离

运行浏览器 MCP 的 Agent 本质上有完整的客户端操作权限。务必在隔离的沙箱或容器中执行，禁止访问内网资源，或者限制允许导航的域名白名单。

## 可复用建议

- **场景拆分**：不要试图让 Agent 一次生成全站测试。逐个页面、逐个流程喂给它，每个任务限定步骤数（如 15 步），质量明显更高。
- **人工门禁**：Agent 生成的测试脚本放入 PR 前需要人工 review，主要检查选择器策略和断言是否有意义。这个过程可以逐步收敛——越用越准。
- **缓存 snapshot**：同一页面如果在多个测试间重复用到，可以缓存初始 snapshot 并作为 prompt 上下文，减少重复浏览器操作和 token 消耗。
- **与 CI 集成**：将 Agent 任务编排进 GitHub Actions，触发条件可以是“前端代码变更合并到主干后”。失败时自动重新运行自我修复流程，并通知到协作工具。

## 总结

Agent 协作不是要替代测试工程师，而是把“从零写脚本→执行→修脚本”的机械劳动自动化。基于 MCP 的浏览器工具让 Agent 真正拥有动手能力，OpenClaw 的 agent 编排则提供了方便的工程接入。当前方案仍需要一定的人工把关，但已经可以显著缩短测试编写的冷启动时间和维护落地成本。如果你已经在用 Agent 做其他自动化，不妨把 E2E 测试也纳入它的能力范围——一旦跑通，它带来的信心是写在代码注释里得不到的。

---

