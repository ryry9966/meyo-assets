---
title: 让 AI Agent 协作写 E2E：从“生成脚本”到“可维护测试”
feedId: 35089
source: 综合讨论
publishedAt: 2026-08-28
---

# 让 AI Agent 协作写 E2E：从“生成脚本”到“可维护测试”

## 背景

E2E 测试的价值没人否认，但维护成本一直是工程团队的长期负担。页面一改版，选择器就失效；手写 Playwright/Cypress 脚本时，还要不断在 DevTools 里确认 DOM 结构和加载时机。LLM 能生成测试代码，但直接生成的脚本往往存在幻觉：选择器脆弱、等待策略缺失、断言不反映业务语义。

真正值得尝试的方向，不是让模型一次性输出测试文件，而是把 Agent 放进“探索—生成—执行—修复”的闭环里。本文以 OpenClaw 挂载 Playwright MCP 为例，介绍一种比较务实的落地方式。

## 问题

直接让通用 Agent 写 E2E，通常会遇到四类问题：

1. **缺少产品上下文**：模型不知道登录流程、权限边界和关键业务状态。
2. **选择器脆弱**：容易生成 `div > div:nth-child(3) > span` 这类绑定实现细节的定位器。
3. **没有真实运行反馈**：一次生成后无法验证，错误定位靠人肉排查。
4. **Agent 有副作用**：自由点击可能触发删除、支付、状态变更等危险操作。

所以，不能把 Agent 当成“自动录屏器”，也不能让它拿着浏览器权限随意操作。

## 做法 / 步骤

### Step 1：拆成两个 Agent

在 OpenClaw 中配置 Playwright MCP，然后按职责拆成两个 Agent：

- **探索 Agent**：只给只读浏览器工具，比如 `navigate`、`snapshot`，负责理解页面结构。
- **执行 Agent**：可以运行 `playwright test`、读取失败报告，并做有限修复。

这样能减少副作用，也让每个 Agent 的上下文更聚焦。

### Step 2：用 ARIA 快照建立元素地图

不要让探索 Agent 直接写测试。先让它用 accessibility snapshot 获取页面的 role、name、label，生成类似下面的定位器：

```ts
page.getByRole('button', { name: '提交订单' })
page.getByLabel('手机号')
```

这类定位器比 CSS class 和 XPath 稳定得多。探索 Agent 还需要记录页面状态：加载完成标志、动态内容、鉴权入口。

### Step 3：生成测试骨架并注入工程约束

给生成阶段的 prompt 明确规则：

- 优先使用 role/label/text 定位器，禁止脆弱 CSS class 和长 XPath。
- 异步操作用 `expect(locator).toBeVisible()`，不要用固定 `sleep`。
- 测试数据用 fixture，不硬编码账号密码。
- 把页面对象和测试 spec 分离开。

同时放 2～3 个项目里已有的好测试作为 few-shot，比纯规则更有效。

### Step 4：运行与受限修复

执行 Agent 运行 `playwright test`，读取失败输出，再驱动修复。这里要限制修复范围：**只允许修改定位器和等待条件，不允许修改业务断言**。

如果断言本身错了，应该交给人来判断，而不是让 Agent 自己改预期结果。

### Step 5：人审与合并

Agent 产出的代码推到临时分支，CI 跑完整测试，人工 review 定位器语义和断言是否反映验收标准。这个步骤不能省。

## 踩坑点

- **副作用失控**：探索 Agent 一旦能点击，就可能误操作。只读模式 + 测试环境是底线。
- **登录态处理**：不要让 Agent 在测试代码里写死密码或 token。用 `storageState` 或环境变量注入。
- **动态加载误判**：Agent 看到 loading 骨架就生成断言，漏掉最终状态。强制它在关键操作后重新快照确认。
- **脆弱 XPath**：模型会倾向复制 DevTools 里的长 XPath。Prompt 禁止 + review 拦截双保险。
- **探索与执行混淆**：Agent 会把调试时点击的状态写进测试。必须分离 explore 和 generate 阶段，生成基于记录而不是实时 DOM。
- **测试数据污染**：创建订单、用户等操作要加随机后缀和清理任务，Agent 不负责数据清理。
- **Token 成本**：全量 DOM 快照很贵。只取 role 树，限制深度。

## 可复用建议

1. **用 ARIA snapshot，不用截图**。文本可解析，定位器更稳定。
2. **维护元素地图 / 页面对象**。让 Agent 只改页面对象，不直接改 spec。
3. **断言业务语义**。例如 `expect(page.getByText('订单已提交')).toBeVisible()`，而不是检查 URL 或 class。
4. **限制修复轮数**。比如最多 3 轮，防止 Agent 在错误路径上死循环。
5. **让 Agent 解释修复差异**。每次改动都要能说明为什么改，便于人审。
6. **单独分支 + CI 验证**。Agent 的产出永远不要直接合入主干。

## 总结

Agent 协作写 E2E 的价值，不是“零人工生成测试”，而是把重复的页面探索、第一轮选择器修复自动化。比较可靠的落点是：探索 Agent 负责建立可维护定位器，执行 Agent 负责跑测和受限修复，人负责业务断言与最终验收。这样一来，E2E 维护的重心会从“补选择器”转向“审上下文”，这才是更可持续的方式。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/bda541a0e39a45a7.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/5ae7c14736ef8ab4.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/c7011562b1f78c8f.png)

