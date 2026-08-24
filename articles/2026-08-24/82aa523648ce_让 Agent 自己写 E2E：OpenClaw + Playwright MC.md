---
title: 让 Agent 自己写 E2E：OpenClaw + Playwright MCP 的闭环实践
feedId: 34455
source: 综合讨论
publishedAt: 2026-08-24
---

E2E 的维护成本通常高于编写成本。选择器漂移、业务流调整、断言弱化，任何一个都会让用例在 CI 里随机变红。最近我们在 OpenClaw 里尝试用 Agent 写 Playwright 用例，最初也翻车：直接生成的代码选择器是猜的，跑不通后又把断言改成空函数，看起来“测试通过”，实际没有任何价值。后来把流程改成“探索-生成-执行-修复”的闭环，效果才稳定下来。核心不是换更强的模型，而是让 Agent 能看见页面、执行测试、读取反馈。

## 环境准备

OpenClaw 中挂载 Playwright MCP。配置大略：

```json
{
  "mcpServers": {
    "playwright": {
      "command": "npx",
      "args": ["@playwright/mcp@latest", "--headless"]
    }
  }
}
```

MCP 暴露 `browser_navigate`、`browser_snapshot`、`browser_click`、`browser_take_screenshot` 等工具。浏览器使用独立 profile，不在生产环境执行。

## 做法

**1. 拆阶段。** 不要一个 prompt 包揽全部。探索 agent 只浏览页面，输出 `exploration.md`；生成 agent 只写测试；执行修复 agent 只读报告和改 diff。

**2. 探索。** 让 Agent 打开目标页面，使用 accessibility snapshot 识别真实角色和标签，选择器优先级固定为 `role > label > testid > text`。禁止在探索阶段生成代码。输出关键步骤和不确定元素。若页面没有 `data-testid`，标注出来让开发补，而不是用绝对 CSS path 硬写。

**3. 生成。** 基于 `exploration.md` 生成 Playwright spec。约束：优先使用 `getByRole` / `getByLabel` / `getByTestId`；登录态通过 fixture 注入；每个用例至少包含一个业务断言，而不只是 URL。例如：

```ts
await page.getByRole('button', { name: '登录' }).click();
await expect(page.getByText('欢迎回来')).toBeVisible();
```

**4. 执行与自修复。** 运行 `npx playwright test tests/e2e/xxx.spec.ts --reporter=json`。失败时读取错误和 trace/screenshot，重新 snapshot 局部页面，再给最小修复，最多 3 轮。修复时要求写 `why.md`，说明根因和改动点，方便 review。

## 踩坑点

- **全量 HTML 会撑爆上下文。** 用 accessibility snapshot 或裁剪后的 DOM，不要让 Agent 直接读整个 HTML。
- **自动修复容易把断言改弱。** 要禁止新增空断言、删除业务验证；diff 必须人工 review。
- **脆弱选择器反复出现。** 生成后可以做一个简单规则检查：命中绝对路径或纯 CSS 定位则打回重写。
- **验证码、第三方登录、支付不可控。** 用测试账号和 mock，不允许 Agent 绕过。
- **副作用控制。** 使用隔离 context，禁止访问真实数据。

## 可复用建议

- 把选择器规范、`data-testid` 命名、修复轮次、禁止行为写成 prompt 模板或规则文件，让每个 agent 读取。
- 测试产物纳入版本控制，首次通过后 review diff，再沉淀为回归用例。
- 失败时用 Playwright trace 作为 Agent 的上下文，比只给报错信息有效。
- 从一条核心链路开始跑通，比如“注册-登录-改资料-退出”，之后再扩展。
- 衡量标准看“首次可用率、修复通过率、人工 diff 行数”，别只看生成了多少测试。

## 总结

Agent 协作写 E2E 的难点不在生成代码，而在建立反馈闭环和约束。OpenClaw 负责编排，Playwright MCP 提供浏览器能力，测试运行器提供结果反馈。先把最小闭环跑通，再用规则收紧行为，AI 生成的测试才可能从“一份初稿”变成“可维护资产”。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/0247dcd511cfaba7.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/034d5e56a2aff440.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/80e02d4dcea73119.png)

