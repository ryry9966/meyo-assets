---
title: 自动化测试的 Agent 协作：把 E2E 从“一次性生成”变成可维护流水线
feedId: 35499
source: 综合讨论
publishedAt: 2026-08-31
---

自动化测试的 E2E 用例维护成本常高于业务代码，界面改版、组件库升级、异步状态变多，都会让选择器快速失效。AI 写测试不新鲜，但单 Agent 从需求直接生成用例，通常只能跑一次。原因是模型不知道真实 DOM、登录态和异步状态，基本在猜选择器。后来我把流程拆成四个小 Agent，在 OpenClaw 里用 MCP 工具串成流水线，产物才稳定可维护。

## 问题：单 Agent 为什么写不好 E2E

单 Agent 典型失败有三类：凭空猜 DOM，写出能过但脆弱的 locator；把登录态、loading、路由跳转混成一团，用 `waitForTimeout` 硬等；修复失败时改断言而不是改实现，让测试失去意义。核心不是模型不够强，而是任务边界不清晰。

## 做法：四段式 Agent 流水线

### 1. Explorer：先摸清真实 DOM
Explorer 使用 Browser MCP 只读能力，载入已保存的登录态，遍历路由抓取 accessibility snapshot，输出 `page_map.json`。每个元素只记 `role/name/testid/input_type/dynamic`，不记 CSS class 和 `nth-child` 结构。

### 2. Scenario：把需求翻译成可执行场景
Scenario Agent 读取 `page_map.json` 和需求文档，输出 Given/When/Then 场景清单。动作必须能映射到页面地图中的元素，不能新造定位方式。空输入、权限失败等边界条件在这一步人工确认。

### 3. Coder：按页面地图生成测试
Coder Agent 按页面地图和场景生成 Playwright 测试。定位器必须来自 `page_map.json`，禁止 `sleep`，等待用 `expect(locator).toBeVisible()` 或 `waitForResponse`。同时生成 `locators.ts` 集中管理定位器，避免全局搜索测试文件。

### 4. Repairer：只修实现，不修语义
Repairer 读取失败 trace 和 DOM snapshot，只允许修三类：定位器失效、瞬时等待、测试代码 bug。断言不成立时标记人工确认，不自动改断言。修复循环最多 3 轮，避免掩盖真实回归。

## 踩坑点

- **登录态没提前准备**：Explorer 抓到的是登录页 DOM，后续全错。先保存 `storageState` 再探索。
- **动态区域未标记**：Repairer 会不断加 timeout，最后变成 sleep 大杂烩。Explorer 应标记 `dynamic` 和需要等待的明确状态。
- **场景语义不能靠修复循环补齐**：边界条件要在 Scenario 阶段人工过，不能等红屏后再补。
- **重复修复成本高**：缓存 `page_map.json`，只对变更路由重新探索，否则 token 和时间都浪费在重复浏览器操作上。
- **Repairer 改动必须输出 patch 和原因**：没有 diff 和理由的自动修复，等于把随机性引入测试库。

## 可复用建议

- 前端统一 `data-testid` 命名，这是给 Agent 最好的接口；没有 testid 时，把页面地图当作文档版本化提交。
- `page_map.json`、场景清单、测试代码三者同源提交，PR 里能直观看到页面变化引起的定位器变化。
- 暂时接不了 Browser MCP 时，先用 Playwright 脚本 dump 每个路由的 a11y tree，让 Explorer 读文件。
- Agent 流水线适合覆盖重复回归路径，新功能的探索性测试仍应保留人工。
- 红线只有一条：机器修定位器，人审业务语义。

## 总结

多 Agent 写 E2E 的核心不是让 AI 变聪明，而是让任务边界清晰：Explorer 回答“页面是什么”，Scenario 回答“验证什么”，Coder 回答“怎么写成代码”，Repairer 回答“红在哪里”。OpenClaw 的 MCP 工具链让这套流水线落地成本很低，产物不再是演示脚本，而是能随页面演进持续维护的测试资产。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/c44c334d29664040.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/be2ca9ed4efd5578.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/bc0039a1e2951bc7.png)

