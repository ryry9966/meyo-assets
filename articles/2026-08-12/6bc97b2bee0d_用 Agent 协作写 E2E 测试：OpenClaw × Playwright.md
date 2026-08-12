---
title: 用 Agent 协作写 E2E 测试：OpenClaw × Playwright × MCP 工程实践
feedId: 32761
source: 综合讨论
publishedAt: 2026-08-12
---

## 背景与痛点

端到端（E2E）测试是质量保障的最后防线，但维护成本高得惊人：UI 一改，定位器失效；流程一长，脚本就像纸牌屋。团队里常有人提“让 AI 写测试”，可落到工程里，不是生成的代码跑不起来，就是断言漏洞百出。我们需要一种**可工程化**的方案：让 Agent 理解页面结构、生成可靠脚本、执行并自愈，而不是只做一次性代码补全。

OpenClaw 社区的 Agent 协作模式和 MCP（Model Context Protocol）的浏览器工具链，恰好提供了一条务实路径。本文记录我们在一个内部后台项目中，用多个 Agent 协同编写并维护 Playwright E2E 测试的真实过程，包含踩坑与可复用设计。

## 整体架构：三个 Agent，一条链路

我们设计了三个 Agent 角色，通过 MCP 服务串联：

1. **Explorer Agent**：负责获取页面 DOM、可交互元素列表和截图，输出结构化的页面描述（JSON 格式）。
2. **Scripter Agent**：接收 Explorer 的输出和测试意图（自然语言），生成 Playwright 脚本，处理定位策略和等待逻辑。
3. **Executor Agent**：执行脚本，捕获异常、截图和日志，反馈给 Scripter 进行修复，形成自愈循环。

通信通过 OpenClaw 的任务管道和 MCP 资源完成。其中 MCP 服务器选用了 `playwright-mcp`，它暴露浏览器操作（导航、点击、截图、获取 DOM）为标准化工具，Agent 可以通过工具的 description 自主选择调用。

## 实践步骤（以登录场景为例）

### 1. 搭建 MCP 浏览器服务
启动 Playwright MCP server，并注册到 OpenClaw 的 MCP 客户端配置：
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
确保 headless 模式能正常运行，且有足够权限访问待测系统。

### 2. 定义 Explorer Agent 的任务
向 Explorer 发送指令：
> 导航到 https://your-app/login，获取当前页面的所有可交互元素（按钮、输入框、链接），输出为以下 JSON 结构：{ "url": "...", "elements": [{"tag": "...", "role": "...", "name": "...", "selector": "..."}] }

Explorer 会调用 `browser_navigate`、`browser_snapshot` 等工具，返回结构化描述。注意，这里不能只给一套固定 selector，而是让 Agent 同时给出多个候选（基于 `data-testid`、`aria-label`、文本内容），方便后续脚本容错。

### 3. Scripter 生成测试脚本
将 Explorer 的输出和自然语言用例一起送给 Scripter：
> 基于以下页面元素描述，编写 Playwright 脚本执行登录测试：输入用户名 admin，密码 pass123，点击登录按钮，验证跳转到 /dashboard 且页面包含“欢迎”。

Scripter 产出的脚本示例：
```typescript
import { test, expect } from '@playwright/test';
test('login', async ({ page }) => {
  await page.goto('https://your-app/login');
  await page.locator('input[name="username"]').fill('admin');
  await page.locator('input[type="password"]').fill('pass123');
  await page.getByRole('button', { name: '登录' }).click();
  await expect(page).toHaveURL(/\/dashboard/);
  await expect(page.locator('h1')).toContainText('欢迎');
});
```
我们会要求 Scripter 优先使用 `getByRole` 或 `data-testid`，而不是脆弱的 CSS 路径，这需要在 prompt 中明确约束。

### 4. Executor 执行与反馈
Executor 调用 `browser_run_code`（playwright-mcp 提供的代码执行工具）运行脚本。若失败，自动截屏、收集错误栈，再次交给 Scripter，附上提示：“登录按钮点击后出现验证码弹窗，请处理。” Scripter 会补充跳过验证码的逻辑（测试环境固定万能码），重新生成代码。

## 踩坑记录

- **动态 DOM 内容获取不全**：Explorer 使用 `browser_snapshot` 时，SPA 的异步加载可能让元素尚未挂载。解决：在 Explorer 指令中加入“等待 3 秒后截图并获取快照，若关键元素缺失则等待额外 2 秒重试”，让 Agent 具备简单的等待策略。
- **多 Agent 上下文丢失**：Scripter 修改脚本后，Executor 再次执行时可能丢失前一轮的运行环境（如会话 cookie）。我们用 MCP 的 `browser_set_cookie` 保持会话，并要求 Executor 在每次重跑前检查登录态，必要时重新登录。
- **生成的断言过于简单**：Agent 倾向于只检查 URL 或文本，忽略关键的业务状态。我们在 Scripter 的 prompt 中嵌入了业务规则列表（JSON 格式），比如“登录后左侧菜单必须包含 3 个指定项”，强制它生成对应断言。
- **MCP server 超时**：长时间脚本执行可能导致 MCP 调用超时。解决方案：将复杂脚本拆分为多个步骤，Executor 分批次执行并返回中间状态，避免单次调用过久。

## 可复用建议

1. **用例模板标准化**：使用 YAML 描述测试步骤，降低 Agent 的理解偏差。例如：
   ```yaml
   test: 用户登录
   steps:
     - navigate: /login
     - fill: { selector: "input[name='username']", value: "admin" }
     - fill: { selector: "input[type='password']", value: "pass123" }
     - click: button[name="登录"]
     - assert: urlMatches /\/dashboard/
   ```
   Scripter 根据 YAML 生成脚本，更可控。

2. **策略分离**：Explorer 负责提取页面信息，Scripter 负责代码生成，Executor 只负责执行和反馈。不要企图让一个全能 Agent 完成所有事，上下文越长越容易出错。

3. **沉淀自愈知识库**：将修复记录结构化存储（错误类型、修复措施）。下次遇到类似错误时，Agent 可先检索历史方案，减少重复尝试。

4. **结合 CI 的闸门机制**：Agent 生成的脚本必须经过一次人工审核才能合入回归套件，避免低质量脚本污染主干。可以在 OpenClaw 工作流中增设确认节点。

## 总结

让 AI 写 E2E 测试，关键不是取代人，而是降低维护成本。通过 OpenClaw + MCP 的 Agent 协作，我们把“页面改动→脚本失效→手动修复”的循环，变为“页面改动→Agent 感知差异→自动建议修补”的半自动流程。当前方案仍需要人类对业务断言的把控，但已经把脚本的机械劳动减少了约六成。后续方向是让 Explorer 与视觉模型联动，更好地理解复杂布局，以及引入更稳定的元素定位策略（如基于视觉锚点的自愈合定位）。

如果在你的项目中也需要这种能力，可以从定义页面的结构化描述协议开始，逐步搭建 Agent 流水线。

---

