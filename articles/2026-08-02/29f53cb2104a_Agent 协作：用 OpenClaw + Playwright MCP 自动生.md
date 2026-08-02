---
title: Agent 协作：用 OpenClaw + Playwright MCP 自动生成可执行的 E2E 测试
feedId: 31357
source: 综合讨论
publishedAt: 2026-08-02
---

## 背景：E2E 测试编写，成本高在哪里？

端到端（E2E）测试是保障核心用户路径不出错的关键手段，但在实际工程中，它往往意味着：

- 大量重复的选择器定位、页面等待、断言编写工作；
- 前端频繁迭代导致选择器失效，维护成本超过编写成本；
- 测试用例的覆盖度依赖人工经验，遗漏交互分支很常见。

“让 AI 直接写 E2E 测试”听起来很诱人，可当你让一个通用 LLM 生成 Playwright 或 Cypress 脚本时，它只能凭空猜测 DOM 结构、动作序列和预期结果。幻觉、脆弱的选择器、缺失的等待逻辑，通常会让生成的代码一次都跑不过。

所以，真正的挑战不是“让 AI 写”，而是“让 AI 看到真实页面，再帮你写”。

这正是 Agent + MCP（Model Context Protocol）的用武之地。

## 问题拆解：AI 写 E2E 测试为什么需要 Agent 协作？

单次生成模式有两个致命缺陷：

1. **脱离真实上下文**：LLM 不知道当前页面有哪些可交互元素，会臆造选择器；
2. **缺乏验证反馈**：生成完就结束了，能不能跑成、断言是否符合预期，全靠人工检查。

Agent 协作的思路是：用多个专业化 Agent，分别负责“探查页面”、“规划交互”、“生成脚本”、“执行并修复”，形成一个闭环流水线。每个 Agent 都通过 MCP 与浏览器环境交互，拿到真实的页面快照，而不是靠记忆里的组件名。

这样，生成的测试脚本不再是一次性彩票，而是经过真实环境验证、接近直接可用的产物。

## 做法：搭建 “探索 → 生成 → 验证” 的 Agent 流水线

这里以 **OpenClaw + Playwright MCP server** 为例，给出一个可复现的最小实践。

### 1. 前置环境

- OpenClaw（开源 AI Agent 框架，支持插件编排）
- Playwright MCP server（提供浏览器控制工具，如导航、截图、获取可访问性树）
- 一个待测 Web 应用（可以是本地开发环境）

将 Playwright MCP 配置为 OpenClaw 的一个工具服务器，使 Agent 可以调用 `navigate`, `snapshot`, `click`, `evaluate` 等能力。

### 2. Agent 角色设计

我们定义三个协作 Agent：

- **Explorer Agent**  
  任务：根据给定的用户故事（如“用户登录后查看订单列表”），在真实页面上逐步探索，记录每一步动作、页面状态（Accessibility Snapshot）和关键元素的角色/名称/测试 ID。
- **Spec Writer Agent**  
  拿到 Explorer 记录的原始交互序列，将其抽象为标准化的测试步骤描述（Given-When-Then 或自然语言操作列表），并补全 UI 断言（例如“页面包含订单编号列”、“登录成功后导航到 /dashboard”）。
- **Code Generator Agent**  
  根据 Spec Writer 的输出，结合 Playwright 模板，生成完整的 TypeScript/JavaScript 测试文件。必须使用 `page.locator` 配合稳定的选择器（优先 `data-testid`、role、label），并显式添加 `waitFor` 逻辑。

最后，一个 **Runner Agent**（可复用 Code Generator 所在环境）负责执行生成的脚本，收集报错，将错误消息和对应行号反馈给 Code Generator，进行最多 3 次自动修复。

### 3. 管道执行流程

在 OpenClaw 中，可以用一个主控 Agent 串起流程：

1. 用户输入目标：“为登录后查看订单列表的流程生成 E2E 测试，URL 是 http://localhost:3000”。
2. Explorer 自动打开无头浏览器，导航到首页，寻找登录入口，输入凭据，点击登录，导航到订单页面，逐步记录 snapshot。
3. Spec Writer 接收 Explorer 输出的 JSON（含步骤和每步的 snapshot 摘要），输出标准化步骤清单和预期结果。
4. Code Generator 生成 Playwright 测试代码，保存到 `tests/order-list.spec.ts`。
5. Runner 执行 `npx playwright test`，捕获失败（如果有），将错误日志重新喂给 Code Generator 修复。
6. 输出最终可用的测试文件。

这个流程可以在本地单次运行，也可以作为 CI 的一个 Job，按需生成新测试或更新已有测试。

## 踩坑点实录

在实践中，有几个非常具体的坑要留意：

- **选择器脆弱性**  
  即使使用 Accessibility Snapshot 拿到了 role 和 name，生成的 `page.getByRole('button', { name: '登录' })` 也可能因为页面中存在多个同名按钮而失败。必须让 Explorer 同时记录上下文（如所在的 section、父级 role），Code Generator 再通过 `.locator` 链或 `.first()` 来消歧。我们最终约束 Code Agent 优先使用 `data-testid`，如果页面上没有，则由 Explorer 尝试用 `evaluate` 注入临时 data-testid 供生成脚本使用——这在测试环境是可接受的。

- **等待与超时**  
  页面加载状态判断是生成的脚本经常漏掉的点。我们要求 Spec Writer 在每一步动作后都明确声明“等待条件”（如 URL 变更、某元素变得可见），Code Generator 必须生成显式 `await page.waitForURL` 或 `waitForSelector`。仅仅依赖 Playwright 自动等待还不够。

- **断言噪声**  
  Spec Writer 容易给出过于宽泛的断言，例如“页面包含订单列表”，生成代码时就变成 `expect(await page.textContent('body')).toContain('订单')`，这样既脆又弱。我们必须限制 Code Generator 只对具体的关键 UI 元素做断言，如“表格行数大于 0”、“筛选器显示正确文案”，并写成稳定的 locator 断言。

- **MCP snapshot 成本**  
  频繁调用 Playwright MCP 的 `snapshot` 会拖慢流程。我们在 Explorer 中做了每步快照的采样（只在新页面或明显 DOM 变化时获取），并通过结构化提示让后续 Agent 不依赖完整的连续截图。

## 可复用建议

如果你想在自己的项目中落地这种模式，以下几点值得内化：

- **从“生成”思维切换到“探索+生成+修复”闭合循环**。单次生成的价值很有限。
- **把测试用例的 spec 与代码生成解耦**。Spec Writer 提供的是环境无关的行为描述，这层抽象使得换测试框架（从 Playwright 到 Cypress）只需要替换 Code Generator，而不是重来一遍。
- **为 Agent 提供工程约定**。例如：项目规范要求所有可交互元素必须有 `data-testid`，那 Agent 生成的选择器会稳定得多。没有约定，Agent 再聪明也难救脆弱的定位器。
- **限制自动修复次数**。无限制的自我修复容易让 Agent 生成越来越怪异的变通代码。设定 3 次上限，如果仍失败，则标记为需要人工介入。
- **将生成流程纳入版本控制**。可以把 Explorer 的记录 JSON、Spec 输出、最终脚本都提交，便于 code review 检测 Agent 的偏差。

## 总结

利用 Agent 协作，我们实际上是在模拟一个有经验的测试工程师的工作方式：先熟悉界面，再拟定测试步骤，再编码，再跑测试，失败后修改。只不过这些现在由 AI Agent 彼此接力完成。

这个方案没有改变测试本身的原则，但显著压缩了从“想测一个流程”到“有一个稳定、可执行的测试文件”之间的机械性劳动。对于频繁迭代的 Web 项目，它可以作为一种按需生成、持续更新的 E2E 测试补充手段。

更进一步，你可以把这些 Agent 集成到开发流程中：当 PR 修改了某个页面，自动触发 Agent 重新探索该页面并更新对应的 specl 文件，实现一种初级的“自愈合测试”。

---

