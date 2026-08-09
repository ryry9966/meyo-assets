---
title: 让 Agent 自己修 E2E 脚本：多智能体协作测试流水线实践
feedId: 32220
source: 综合讨论
publishedAt: 2026-08-09
---

# 让 Agent 自己修 E2E 脚本：多智能体协作测试流水线实践

## 背景：写 E2E 测试的“最后十公里”

E2E 测试的收益不用多说，但维护成本高一直是团队的痛点。UI 调整、页面重排、前端状态异步加载，都会让原本跑得通的脚本突然失败。团队花在“修测试”上的时间，往往比写新测试还多。

最近半年，随着 Playwright MCP、OpenClaw 这类将浏览器工具协议化的基础设施成熟，大家对“用 AI 写测试”的想象开始落地。但实际用下来会发现，单靠一个大模型一把梭生成 Playwright 脚本，产出的代码基本只是“看起来不错”，真正在 CI 里稳定运行的不到一半——定位器脆弱、缺少等待逻辑、断言粗放，都是老问题。

这篇文章想探讨的不是“AI 帮你生成测试用例”，而是如何用**多个 Agent 分工协作**，让测试从“生成完就扔”变成“生成、执行、自我纠错、再验证”的闭环。我们基于 OpenClaw 的插件生态和 MCP Server 能力，搭了一条能用、可复用的轻量级流水线。

## 问题拆解：单一 Agent 为什么不够

用一个 Agent 做 E2E 生成时，通常会这样提示：

> 你是一个测试工程师，请用 Playwright 为登录页写一个测试脚本。

模型会输出一段代码，但它缺乏页面真实结构的感知（哪怕给了 HTML 片段，也只是静态快照），更糟糕的是，生成后没有验证回路。脚本能不能跑通、元素是不是可见、异步接口有没有完成，生成 Agent 完全不管。

所以我们需要把任务拆成三个角色：

- **Explorer Agent**：负责实时探索页面，输出结构化的元素地图。
- **Author Agent**：基于元素地图和测试意图，生成稳定的测试脚本。
- **Executor & Reviewer Agent**：执行脚本，捕获失败现场，把诊断信息回传给 Author 修正。

这三个 Agent 共享同一套工具（Playwright MCP），但各自有独立的系统提示和目标，通过 OpenClaw 的任务编排串联起来。

## 搭建步骤：从环境到回路

### 1. 环境准备

需要启动一个 Playwright MCP Server，并确保 OpenClaw 能调用其中的工具。最小配置如下（`mcp.json`）：

```json
{
  "mcpServers": {
    "playwright": {
      "command": "npx",
      "args": ["-y", "@anthropic/mcp-server-playwright"]
    }
  }
}
```

启动后，OpenClaw 的 Agent 即可通过 `mcp__playwright__browser_navigate`、`mcp__playwright__browser_snapshot` 等工具操控真实浏览器。

### 2. Explorer Agent：生成元素地图

第一个 Agent 的职责是理解页面。它的系统提示强调只做探索，不生成操作代码：

> 你是一个页面探索助手。请导航到目标地址，使用 browser_snapshot 获取完整的可访问性树，然后提炼出所有可交互元素及其语义标识（role、name、data-testid、placeholder等）。输出为一个结构化 JSON 数组，每个元素必须包含稳定定位策略（优先 data-testid，其次 role+name，避免 class 和绝对 xpath）。

运行完成后，我们得到一份类似这样的元素地图：

```json
[
  {
    "label": "用户名输入框",
    "locator": "input[data-testid='username']",
    "role": "textbox"
  },
  {
    "label": "密码输入框",
    "locator": "input[type='password']",
    "placeholder": "请输入密码"
  }
]
```

这份地图会作为上下文注入给下一个 Agent。

### 3. Author Agent：生成带断言的测试脚本

Author Agent 拿到元素地图和测试意图（如“正常登录流程”），生成 Playwright 代码。关键提示点：

- 强制使用元素地图中推荐的定位器。
- 要求为每个关键步骤添加显式等待（`waitForSelector` 或 `expect(locator).toBeVisible()`）。
- 生成代码后不执行，直接返回。

产出示例：

```javascript
const { test, expect } = require('@playwright/test');
test('normal login', async ({ page }) => {
  await page.goto('https://example.com/login');
  await page.fill('input[data-testid="username"]', 'admin');
  await page.fill('input[placeholder="请输入密码"]', '123456');
  await page.click('button[data-testid="login-btn"]');
  await expect(page.locator('.dashboard')).toBeVisible({ timeout: 10000 });
});
```

### 4. Executor & Reviewer Agent：执行、诊断、纠错

这是整个流水线中最关键的环节。我们不再由人类去手动跑一遍脚本然后改代码，而是让一个 Review Agent 干这件事。

Review Agent 的系统提示核心逻辑：

> 运行给定的 Playwright 脚本，如果执行成功，输出最终截图并确认通过。如果失败，捕获错误信息、当前页面可访问性树及页面截图，分析失败原因：1) 元素定位失效？2) 超时？3) 断言不符？将诊断报告回传给 Author Agent，并要求重新生成代码。重试不超过 3 次。

**反馈回路**：Author 收到诊断后，会看到类似“定位器 `input[data-testid='username']` 在页面中未找到，当前页面按钮文本已变更为‘Sign In’”这样的信息，它就能有针对性地修复脚本，比如改用 `button[type='submit']` 或根据新 snapshot 调整。

通过 OpenClaw 的多轮对话上下文，三个 Agent 的“交谈”自然保持连续性，所有中间产物（地图、脚本、错误报告）都留在同一 session 中，无需外部存储。

## 踩坑记录

### 异步状态与快照不一致
Explorer 在 `networkidle` 后抓取 snapshot，但动态渲染的组件可能稍后出现，导致生成的地图缺少元素。解决方案：在 snapshot 前显式 `waitForLoadState('networkidle')` 并额外等待 500ms，或针对特定选择器等待可见性。

### 定位器过于智能反而脆弱
Author 有时会自作主张用 `getByRole` 生成的复杂选择器，例如 `page.getByRole('button', { name: /login/i })`。在中文环境或文本变化时极易失效。我们的强制约束是：**所有生成脚本必须优先使用数据属性，如果不存在，则使用 input/button 的原生属性（type、placeholder）**，并在提示中给出反例。

### 工具限流与重试风暴
Playwright MCP 在高频调用下可能触发浏览器无响应。Review Agent 在失败回传时必须显式检查错误类型，如果是连接错误，应直接中止并提示人工介入，而不是无脑重试。

### 上下文污染
如果一条 session 里同时处理多个测试场景，前面页面的状态可能残留。我们约定每个测试流程在独立 session 中运行，或者每次重新导航清除状态（`context.clearCookies()`）。

## 可复用建议

这套流水线完全可以封装成一个 OpenClaw 插件或一个组合 MCP 工具。我们的做法是：

- 将 Explorer / Author / Reviewer 的提示模板、错误处理逻辑固化为一个“E2E Agent Workflow”配置。
- 暴露一个简单入口命令，例如 `@e2e-bot test login flow https://example.com/login`，内部自动调度三个 Agent。
- 把最终通过的脚本和截图输出到预设目录，方便直接接入 CI。

如果你的团队已经在用 Playwright，可以把生成的脚本直接扔进现有项目，搭配已有的 fixture 和配置，AI 生成的代码就当是一个自动补丁，不需要重构整个测试体系。

## 总结

Agent 协作并不是把一个大 prompt 拆成三个那种形而上的优化，而是让每个角色能各司其职，特别是引入“执行-反馈”回路后，测试脚本的可用性显著提升。在我们的内部实验里，经过三轮以内纠错，登录、注册、列表查询等典型流程的通过率从单 Agent 的约 40% 提升到 85% 以上。剩下的失败原因多是第三方服务不稳定或验证码这类硬骨头，这些本来也不该交给自动化死磕。

如果你现在正被 E2E 脚本的维护折磨，不妨开始用 MCP 打通浏览器，然后逐渐把修复脚本的工作交给 Agent，你的角色从“修测试的人”变成“审阅测试结果的人”，这在工程上是完全可行的。

---

