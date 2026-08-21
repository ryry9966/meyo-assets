---
title: Agent 辅助 E2E：从 Playwright MCP 到可维护测试的实践
feedId: 34036
source: 综合讨论
publishedAt: 2026-08-21
---

## 背景

E2E 测试是回归的兜底，但维护成本很高：选择器失效、业务流变更、测试数据准备繁琐、报错信息不直观。通用模型生成测试代码时，经常像“背题”——看起来语法正确，跑起来全挂，或者只写了几个无关痛痒的断言。

真正有价值的场景，不是让 AI 凭空写测试文件，而是让它参与一个闭环：**探索页面 → 生成用例 → 运行验证 → 修复失败**。这需要 Agent 具备操作浏览器、读取 DOM、执行测试和读取报错的能力。

如果你已经在 OpenClaw 里挂载 MCP，可以把 Playwright MCP 作为浏览器工具节点，让 Agent 不只生成代码，还能像测试工程师一样操作页面。

## 问题

直接让 Chat 写 Playwright 用例，通常会遇到四个问题：

1. **不了解真实页面结构**：选择器靠猜，生成的 `page.click('.btn-primary')` 在真实项目里可能命中多个元素或直接失效。
2. **不了解登录态、权限和路由**：用例从错误的入口开始，甚至跑不起来。
3. **无法运行验证**：模型输出一次代码后，无法自己发现运行时报错。
4. **断言质量差**：要么过弱，比如只断言页面标题；要么过强，依赖脆弱的文案或 DOM 顺序。

## 做法：让 Agent 参与“探索-生成-运行-修复”闭环

### 1. 先建立测试契约

在代码里统一加 `data-testid`，并写清规则：

- 只允许用 `data-testid` 定位；
- 路由和权限状态要可跳转；
- 测试账号、夹具由统一的 fixture 创建；
- 禁止使用 `waitForTimeout` 硬等待。

把这份契约放进 Agent 上下文，或者做成 MCP 资源，每次生成用例前强行加载。

### 2. 让 Agent 先探索真实页面

不要让它直接写文件。先给一个探索任务：

> 从 `/login` 开始，用测试账号登录，打开目标页面，记录关键区域的 URL、按钮 testid、可见文本和可访问性树。

这样生成的用例基于真实 DOM，而不是模型记忆。比如 Agent 会知道“创建项目”按钮的 testid 是 `create-project`，而不是猜一个 `.btn-add`。

### 3. 生成测试并立即运行

用固定模板生成 Playwright 测试，例如：

```ts
test('can create project', async ({ page }) => {
  await page.goto('/projects');
  await page.getByTestId('create-project').click();
  await page.getByTestId('project-name').fill('agent-demo');
  await page.getByTestId('submit').click();
  await expect(page.getByTestId('project-card')).toHaveText('agent-demo');
});
```

生成后立即通过 MCP 或 CLI 执行。把报错信息原样回给 Agent，让它最多修复 3 轮。只要超过 3 轮，就交给人介入，避免越改越乱。

### 4. 设置运行边界

Agent 操作浏览器时，要明确禁止点击支付、删除、权限变更等危险操作。对于可能产生真实副作用的按钮，改用 mock 接口，或者要求先在沙箱环境运行。

## 踩坑点

- **页面快照过大**：可访问性树如果全量塞给模型，token 很快耗尽。只保留可见元素、testid 和 role，先裁剪再过模型。
- **把 `waitForTimeout` 当万能药**：Agent 遇到时序问题容易硬等。明确禁止 `waitForTimeout`，统一用 `expect(...).toBeVisible()` 或等待接口响应。
- **环境漂移**：MCP 浏览器和 CI 浏览器的视口、locale、storageState 不一致，导致用例在本地过、CI 挂。固定 viewport，把登录态保存为 `storageState.json` 复用。
- **生成用例通过但没价值**：只断言“页面有标题”没有意义。要求关键业务结果必须断言，例如创建后列表出现新条目、状态变更后刷新仍存在。
- **Agent 越改越乱**：修复轮数不要超过 3，超过后立即停止，保留现场日志，由人判断是测试问题还是业务缺陷。

## 可复用建议

- 把测试契约、代码模板和禁止项做成一版 Prompt 或 MCP 配置，新项目直接复用。
- CI 分成两段：`test:gen` 生成到临时分支，`test:run` 在 PR 中执行真实用例。生成分支不直接合并到主干。
- 保留每次运行的截图或录像。失败时让 Agent 先读截图和日志，再决定是否修改用例，而不是盲目重跑。
- 优先覆盖核心流程，不要追求全量覆盖。一个稳定用例胜过十个脆弱用例。

## 总结

Agent 写 E2E 的价值不在“自动产生代码”，而在把人工看页面、试选择器、跑报错循环这些昂贵动作交给工具。你只需要保留两件事：测试契约和最终 review。当 AI 能操作浏览器、读 DOM、运行测试并修复失败时，它才从“补全工具”变成真正的测试协作者。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/91f619feecc65bd7.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/108c32dd13068c51.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/54522d98b3fa5d33.png)

