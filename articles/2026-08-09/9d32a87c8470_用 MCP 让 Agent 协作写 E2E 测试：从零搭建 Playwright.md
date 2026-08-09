---
title: 用 MCP 让 Agent 协作写 E2E 测试：从零搭建 Playwright 测试生成流水线
feedId: 32211
source: 综合讨论
publishedAt: 2026-08-09
---

# 用 MCP 让 Agent 协作写 E2E 测试：从零搭建 Playwright 测试生成流水线

## 背景：E2E 测试的维护债

但凡维护过前端项目的人，都对 E2E 测试又爱又恨。爱的是它能拦住回归问题，恨的是它的编写和维护成本。一个登录流程的测试脚本，从定位元素、处理异步、写断言，到适配不同环境，熟练的工程师也要花 20 分钟。一旦页面改版，脚本大面积失效，往往需要逐条排查、手动修定位符。

市面上的“录制回放”工具或者 AI 直接生成代码的尝试，大多停留在 demo 阶段，生成的脚本硬编码坐标、XPath，换一个分辨率就崩。我们需要的不只是代码生成器，而是一个能 **理解上下文、操作页面、并产出可维护测试脚本的协作系统**。这正是 Agent + MCP 的用武之地。

## 问题：AI 直接写测试为什么不靠谱？

如果你直接把页面截图丢给大模型让它“写个 Playwright 测试”，结果大概率是：
- 捏造不存在的元素 id；
- 写错等待逻辑，脚本秒崩；
- 无法处理登录态、权限等业务流程。

根本原因在于模型缺少与浏览器的**实时交互通道**。它只能靠静态信息猜测，猜错是常态。

## 方案：Agent 协作式测试生成

我们采用 **OpenClaw Agent + Playwright MCP Server** 的架构，让模型以工具调用的方式操纵真实浏览器，边操作边思考，最终生成一份经过验证的测试脚本。

整体流程：
1. **Agent 接到自然语言任务**，例如“为订单列表页的筛选功能生成 E2E 测试”。
2. **Agent 通过 MCP 工具**导航到目标页面，获取无障碍树或 DOM 快照。
3. **Agent 尝试执行操作**（点击、输入），观察页面变化，定位关键元素。
4. **Agent 生成 Playwright 测试代码**，并在同一个浏览器会话中试运行，验证通过性。
5. **产出脚本**，提交到项目仓库。

核心是 **“先探索再生成，生成后自测”** ，形成闭环，大幅减少人工试错。

## 实践步骤

### 1. 启动 Playwright MCP Server

使用社区提供的 `playwright-mcp-server`，它暴露浏览器操作工具：导航、截图、点击、输入、执行 JS 等。以 Node 为例：

```bash
npx @anthropic/mcp-server-playwright
```

OpenClaw 客户端配置对应的 MCP 连接，让 Agent 可直接调用 `browser_navigate`、`browser_click`、`browser_snapshot` 等工具。

### 2. 定义 Agent 的任务与约束

编写一个系统提示，限制 Agent 的输出格式和代码风格：

```text
你是一个 E2E 测试工程师。请根据用户描述的场景，操作浏览器并生成 Playwright 测试代码。
规则：
- 优先使用 data-testid 选择器，找不到时使用无障碍标签。
- 所有异步操作必须使用 await 和适当等待。
- 生成的代码必须包含完整的 test() 块和 expect 断言。
- 生成后必须在当前浏览器中试运行，若失败需修复。
```

用户只需给出业务描述：“在商品列表页，点击‘价格升序’按钮后，验证第一个商品价格 ≤ 第二个商品价格”。

### 3. Agent 探索与生成

Agent 会依次调用：
- `browser_navigate` 进入列表页
- `browser_snapshot` 获取可交互元素列表
- `browser_click` 点击排序按钮
- 再次抓取快照，提取排序后的价格文本
- 生成代码并在内置的 `playwright` 环境中执行

典型产出如下：

```typescript
import { test, expect } from '@playwright/test';

test('sort by price ascending', async ({ page }) => {
  await page.goto('/products');
  await page.click('[data-testid="sort-price-asc"]');
  await page.waitForSelector('.product-item');
  const prices = await page.$$eval('.product-price', els => 
    els.map(el => parseFloat(el.textContent.replace('¥','')))
  );
  for (let i = 1; i < prices.length; i++) {
    expect(prices[i]).toBeGreaterThanOrEqual(prices[i-1]);
  }
});
```

### 4. 人工审查与入库

Agent 生成的脚本放在 `__tests__/e2e-generated/` 目录下，由开发者审查：
- 确认选择器是否合理，是否过度依赖脆弱属性；
- 补充边界条件；
- 纳入常规测试套件。

### 5. CI 集成与自动修复

把 Agent 能力接入 CI：当开发分支的页面结构变化导致测试失败时，可触发 Agent 自动尝试修复定位符，并提交 PR。这一步需谨慎，只建议在 **预发布分支** 上启用，并强制人工审核。

## 踩坑点

- **元素定位不稳定**：Agent 有时会选用 `aria-label` 或文本内容做选择器，一旦文案调整就挂。强制命令优先使用 `data-testid`，并在提示中明确优先级，能显著提升鲁棒性。
- **等待时机不当**：生成代码容易遗漏等待，导致 flaky 测试。我们让 Agent 在生成后试跑三次，三次都通过才输出，过滤掉大部分时序问题。
- **状态污染**：测试间共享浏览器上下文可能产生副作用。建议每次生成新测试时重置存储状态或使用全新上下文。
- **Agent 幻觉**：模型会“编造”验证步骤，比如断言一个不存在的 toast 文案。通过限定只用快照中可见的内容写断言，可有效减少幻觉。

## 可复用建议

1. **建立专属提示词库**：针对不同页面模式（表单、列表、详情）准备不同的生成约束，Agent 按需调用。
2. **抽象测试模板**：将登录、设置权限等公共步骤封装为 fixture，让 Agent 生成的测试复用这些 fixture，提升可读性。
3. **记录常见修复模式**：当一个定位符因为页面改动从 #login-btn 变为 [data-testid="login"]，模型修复后可将这种映射规则记录为知识，供后续使用。
4. **版本控制与对比**：每次生成或修复的测试都保留 diff，方便回溯和发现 Agent 的“坏习惯”。

## 总结

让 Agent 直接写 E2E 测试不是异想天开，但也不可能取代人工审查。通过 MCP 赋予它真实浏览器操作能力，结合合理的约束与闭环自测，可以把繁琐的测试编写工作变成一句话描述即可交付初稿。我们将这种方式投入实际项目的回归测试维护中，新功能测试的初始脚本编写时间平均缩短了 60%，而稳定性在几轮调优后也与手写脚本持平。如果你已经在用 OpenClaw 做自动化，不妨花一个下午搭一套 Playwright MCP 流水线，测试团队的反馈可能会让你惊喜。

---

