---
title: 用 OpenClaw Agent 协作自动生成 E2E 测试：从 Playwright MCP 接入到可持续维护
feedId: 32075
source: 综合讨论
publishedAt: 2026-08-08
---

## 背景：E2E 测试的“维护债”

端到端测试是保障核心链路最直接的手段，但编写和维护成本极高。页面一改版，选择器就失效；异步逻辑稍复杂，等待策略就要重调；更头疼的是，业务迭代快，测试用例永远追不上功能变化。

过去一年，Agent 协作模式在自动化领域逐渐成熟。如果能让 AI Agent 直接操控浏览器、理解页面结构，再根据自然语言生成并更新测试脚本，就能把大量重复劳动转化为审核与微调。这不再只是“代码补全”，而是 Agent 用工具链协助完成整个测试生命周期。

OpenClaw 作为开放 Agent 框架，通过 MCP（Model Context Protocol）可以对接浏览器自动化工具，使这一流程工程化落地。下面以 Playwright MCP 为例，说明如何让 Agent 写 E2E 测试，以及过程中真正需要注意的工程问题。

## 核心思路

让 Agent 编写 E2E 测试，不是简单问一句“给我生成登录页测试”，而是让它完成以下步骤：

1. 通过 MCP 工具控制浏览器访问目标页面
2. 结构化地获取 DOM 元素、数据属性、可访问性角色
3. 理解页面行为，生成一个完整的 Playwright 测试文件（包括断言和等待）
4. 执行测试，根据失败信息自我修正，直到通过

这一步的背后，Agent 需要把“页面快照”转换为“可靠的定位策略”，这是最大的难点，也是踩坑最多的地方。

## 接入步骤

### 1. 启动 Playwright MCP Server

首先按 OpenClaw 插件规范配置 Playwright MCP，示例 `mcp.json`：

```json
{
  "mcpServers": {
    "playwright": {
      "command": "npx",
      "args": ["@anthropic/mcp-server-playwright", "--headless", "--browser=chromium"]
    }
  }
}
```

确保 OpenClaw 工作区加载该配置，Agent 就可以调 `browser_navigate`、`browser_snapshot`、`browser_click` 等工具。

### 2. 设计测试生成的 Prompt

需要给 Agent 明确的约束，否则生成的代码可用性很差。常用指令模板：

> 你是一个 E2E 测试工程师。使用 Playwright MCP 工具完成以下任务：
> - 打开 https://example.com/login
> - 获取页面快照，优先使用 data-testid、role、text 作为定位器，禁止使用脆弱 CSS 类名或 XPath
> - 生成一个独立的 Playwright 测试文件，包含 `test.describe`，覆盖率需包含正常登录、空字段校验、错误密码提示
> - 运行 `npx playwright test` 验证，如果失败根据错误信息修改测试，最多尝试 3 次

这样 Agent 会先探索页面，提取语义化选择器，再生成测试并自测。

### 3. 示例交互过程

Agent 会先调用 `browser_navigate` 打开页面，然后调用 `browser_snapshot` 得到可访问性树：

```
- heading "Sign in" [level=1]
- textbox "Email" [data-testid="email-input"]
- textbox "Password" [data-testid="password-input"]
- button "Log in" [data-testid="login-button"]
```

基于此生成类似代码：

```ts
import { test, expect } from '@playwright/test';

test.describe('Login', () => {
  test('successful login', async ({ page }) => {
    await page.goto('/login');
    await page.getByTestId('email-input').fill('user@example.com');
    await page.getByTestId('password-input').fill('correct-password');
    await page.getByTestId('login-button').click();
    await expect(page).toHaveURL('/dashboard');
  });
});
```

随后 Agent 可执行 `npx playwright test`，将终端输出作为反馈，修改失败用例。

## 踩坑点与对策

**坑1：页面状态不稳定，快照提取不全**

很多现代应用会用骨架屏、渐变加载。Agent 在第 3 秒和第 6 秒获得的快照结构可能完全不同，导致生成的选择器无法回放。

对策：让 Agent 在关键交互前显式调用 `browser_wait_for`（等待特定文本或元素出现），并在 Prompt 中要求“若快照缺少预期元素，先等待 2 秒重新快照”。

**坑2：生成的代码过度依赖即时快照**

Agent 看到 `data-testid="submit-btn"` 就只会用 `getByTestId`。一旦前端去掉该属性，测试全部崩溃。

对策：在 Prompt 中注入防御性选择策略优先级：`data-testid` > `role` + `name` > `text` > `placeholder`。同时要求 Agent 在同一元素上提供 2-3 个备用定位器，以注释形式写入测试，方便人工快速替换。

**坑3：自动修复陷入死循环**

如果页面本身有 bug（如登录失败无错误提示），Agent 会反复修改断言和等待，3 次尝试后抛出一个奇怪的长代码。真实的 CI 中这是噪音。

对策：设定最大重试次数并记录每次失败的错误信息，超过阈值直接将完整日志交给人审阅，不继续生成。

**坑4：生成测试不维护数据隔离**

E2E 测试经常依赖特定状态（需注册用户、特定订单）。Agent 生成的快乐路径测试可能污染线上数据或需要预置数据。

对策：要求 Agent 生成的测试必须包含 setup 部分（如 API 创建测试用户）和 teardown，并显式标注“需要 `TEST_ENV=true` 前提”。

## 可复用建议

- **抽象“测试技能”为 OpenClaw 技能包**：将经过验证的 Prompt 模板、定位策略、等待模式封装为可加载的 skill，例如 `e2e-login-skill`，团队共享。
- **配合 Page Object 模式落地**：让 Agent 生成的测试不直接操作 `page`，而是生成 PO 类。Prompt 中加入：“先探索页面，生成对应的 Page Object 类，然后基于 PO 编写测试”。
- **与现有测试套件对接**：Agent 输出的测试文件应自动放到约定目录（如 `tests/e2e/`），避免手动搬移。
- **用版本 diff 做审核门槛**：每次 Agent 更新测试后生成一个 `git diff` 概览，团队只审核变更部分，而不是全部重读。
- **关注 MCP 工具链的升级**：Playwright MCP 的 `browser_evaluate` 可用于获取动态渲染后的 DOM，但要注意安全。在受控环境内开放该工具，可以大幅增强页面探索能力。

## 总结

让 Agent 协作编写 E2E 测试，已经从“玩具级演示”发展为可以工程化的实践。借助 OpenClaw 的 MCP 插件体系，Agent 能够直接操控浏览器、提取可访问性信息并生成有意义的测试，甚至完成初步的自修复。

但它的正确定位不是“替代测试工程师”，而是“将机械化的选择器编写和简单用例生成自动化”，让人专注于业务逻辑校验与异常场景设计。做好页面状态等待策略、定位器优先级和人工审核卡点，是落地的三个核心前提。

在接下来的迭代中，可以把 Agent 生成的测试加入 CI，让它像“轮值测试助理”一样维护基线用例，逐步降低回归测试的维护成本。

---

