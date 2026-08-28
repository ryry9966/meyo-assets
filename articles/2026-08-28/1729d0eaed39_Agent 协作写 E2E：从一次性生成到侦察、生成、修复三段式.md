---
title: Agent 协作写 E2E：从一次性生成到侦察、生成、修复三段式
feedId: 35029
source: 综合讨论
publishedAt: 2026-08-28
---

## 背景
E2E 测试的痛点不是“不会写”，而是“写出来之后要长期维护”。不少团队尝试用 AI 生成 Playwright/Cypress 用例，但发现生成的用例在本地能跑通，进了 CI 就频繁失败，或者选择器脆弱到前端改一个 class 就挂一片。问题不在模型能力，而在任务定义过于粗糙：把需求描述丢给一个 Agent，让它直接输出完整测试，既缺少真实 DOM 信息，也没有验证反馈。

## 问题
单 Agent 一次性生成 E2E 通常会踩三个坑：

- 不了解真实页面结构，选择器靠猜，常用 `.btn-primary`、`text=登录`、`nth-child(3)`。
- 断言策略激进，把不稳定的 UI 状态当成业务事实，比如断言某个 loading 文案消失。
- 缺乏运行反馈，生成即结束，没有闭环修复。

结果就是测试代码本身成为新的维护负担。

## 做法：三个 Agent 协作
在 OpenClaw 里，我会把“写 E2E”拆成三个可独立执行的子任务，共用同一个 browser MCP 工具。

### 1. 侦察 Agent
只负责观察，不负责写测试。它通过 MCP 打开目标页面，读取可访问性树、路由信息、关键交互元素、现有 `data-testid`。输出结构化的页面说明，例如：

- 页面路由：`/login`
- 关键元素：邮箱输入框、密码输入框、登录按钮
- 可用选择器：`getByTestId('login-email')` 缺失，建议补充
- 交互路径：输入邮箱 → 输入密码 → 点击登录 → 跳转 `/dashboard`

这一步把不确定的 DOM 结构变成可被后续 Agent 消费的有限上下文。

### 2. 生成 Agent
接收侦察结果和验收标准，按模板生成 Playwright 用例。必须用约束 prompt 锁死选择器优先级和等待策略。例如：

```text
你只能使用 page.getByTestId / page.getByRole / page.getByLabel。
禁止使用 CSS class、绝对 XPath、nth-child。
禁止使用 page.waitForTimeout，必须使用 expect(locator).toBeVisible()。
```

同时要求生成用例时，把登录态从 `beforeEach` 中导入，不在测试体内硬编码 token/cookie。

### 3. 验证修复 Agent
运行生成的测试，读取失败栈和 DOM 快照，做有限修复：只允许改选择器、等待策略、测试数据，不允许改业务断言。每轮修复后重跑，最多 3 轮；超过 3 轮仍失败，输出 diff 和失败原因，转人工处理。

这个闭环可以把“生成即结束”变成“生成-运行-修复-再运行”的工程流程。

## 踩坑点
- **上下文爆炸**：把整页 HTML 交给模型不仅慢，还会让模型拟合无关结构。侦察 Agent 只保留可访问性树和关键片段，必要时截断。
- **模型仍然会写脆弱选择器**：即使 prompt 约束了，也偶尔出现 `.login-btn`。验证 Agent 跑完后要有静态检查，优先用 `getByTestId` 规则扫描。
- **自动修复别越界**：失败可能是业务逻辑变了，Agent 为了“跑绿”可能改断言。必须限制修改范围，且最终 diff 必须人工 review。
- **不要硬编码登录态**：登录流程要抽离成全局 setup 或 fixture，否则每个用例都在重复登录，维护成本翻倍。
- **动态内容用自动等待**：避免 `sleep(2000)`，用 `expect.poll` 或 `toBeVisible` 处理异步。

## 可复用建议
- 维护一份测试知识库：路由表、角色权限、常用 fixture、选择器约定，让生成 Agent 先读再写。
- 从冒烟路径开始：先覆盖 5 到 10 条核心用户流程，跑通并稳定后再扩展。
- 用 MCP 限权：只给 Agent 暴露浏览器操作和测试运行命令，不给任意 shell。
- 建立 testid 回填循环：侦察 Agent 发现关键元素缺 `data-testid` 时，输出前端补丁建议，逐步降低选择器脆弱性。
- 保留 CI 门禁和人工 review：AI 生成的测试必须通过 CI，且 PR 中要能看到 Agent 的修改 diff。

## 总结
Agent 协作写 E2E 的价值，不在于单次生成多惊艳，而在于把工程步骤拆开：侦察降低不确定性，生成套用项目约束，修复建立反馈闭环。这样产生的用例更接近“可维护的回归资产”，而不是“一次性生成的摆设”。在 OpenClaw 里，合理编排子 Agent 和 MCP 工具，能把这个流程跑成半自动化，但最终门禁仍需人来把关。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/ac8d2b6844712757.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/23fe72a71764bab8.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/36abaa2a90f690e6.png)

