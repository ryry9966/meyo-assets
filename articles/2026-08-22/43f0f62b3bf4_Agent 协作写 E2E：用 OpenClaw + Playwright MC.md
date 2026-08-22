---
title: Agent 协作写 E2E：用 OpenClaw + Playwright MCP 搭一条可落地的测试流水线
feedId: 34181
source: 综合讨论
publishedAt: 2026-08-22
---

## 背景
E2E 测试的维护成本通常高于单测。问题不在“写脚本”，而在页面一改，选择器批量失效；断言只检查文案，业务行为没被验证。引入 AI 后容易走到另一极端：让一个 Agent 直接根据需求生成 Playwright 脚本。结果往往是选择器靠猜，失败后无法判断是测试脚本错还是产品回归。

## 问题
单个 Agent 写 E2E 有三个典型失败模式：

1. **脱离真实页面**：它读组件源码或截图，生成 class 选择器，但真实页面是异步渲染、动态列表。
2. **失败不可复现**：CI 里只留下 `element not found`，没有 DOM 快照、可访问性树或 console 错误。
3. **自动修复不可控**：失败后让 Agent 直接改测试，可能把真实业务 bug 修成“期望错误结果”。

## 做法：拆成 5 个角色
在 OpenClaw 里我拆成 Planner、Explorer、Writer、Runner、Triage，共享一个 Playwright MCP server。MCP 只暴露少量工具：

- `browser_navigate`
- `browser_snapshot`
- `browser_click`
- `browser_type`
- `browser_assert`
- `browser_wait_for`
- `browser_read_console`

**Planner** 负责把业务路径拆成最小步骤，例如：登录 → 创建项目 → 修改状态 → 断言列表更新。每个步骤必须有可验证的中间状态。

**Explorer** 负责打开页面并抓取 ARIA snapshot，输出 role/name/可交互元素，而不是让 Writer 直接读 HTML 猜 class。

**Writer** 只根据 Planner 的步骤和 Explorer 的 snapshot 生成测试代码。定位方式优先 `getByRole` / `getByLabel` / `data-testid`，禁止从源码拼 CSS path。

**Runner** 执行测试。失败时自动抓取失败步骤前后 3 秒的 DOM snapshot、console error 和网络错误，交给 Triage。

**Triage** 输出结构化 JSON：

```json
{
  "failure_type": "SELECTOR_STALE",
  "confidence": 0.92,
  "evidence": "button role=button name=保存 已从 header 移至 footer",
  "suggested_patch": "use getByRole('button', { name: '保存' })"
}
```

失败类型分成四类：`SELECTOR_STALE`、`ASSERTION_OUTDATED`、`ENV_ISSUE`、`POSSIBLE_BUG`。只有前两类允许自动 patch，`POSSIBLE_BUG` 必须转人工确认。

## 踩坑点
- **别让 Agent 直接读 HTML 生成 CSS path。** ARIA snapshot 更短，也更接近真实用户行为。
- **固定 sleep 是坑。** 生成代码里出现 `waitForTimeout` 要拦截，替换成显式等待或 `browser_wait_for`。
- **MCP 工具给得太全，Agent 会乱试。** 限制到 6-8 个，禁用任意 JS 执行类工具。
- **页面快照 token 消耗很大。** Explorer 要截断：忽略不可交互节点，只保留 role/name/状态。
- **自动修复要加边界。** 只允许改选择器或等待条件，不允许改断言期望值，否则会掩盖业务 bug。

## 可复用建议
- 维护一个 `e2e-conventions.md`，记录 `data-testid` 约定、登录态复用和已知动态区域，Writer 每次先读。
- 把 Triage 输出落成 CI artifact，例如 `triage.json`，用于统计“测试问题多”还是“产品回归多”。
- 用 OpenClaw 插件封装成 `e2e:generate`、`e2e:run`、`e2e:triage` 三个命令，PR 里可直接触发。
- 登录态用 storageState 或会话复用，避免每个用例重复登录。
- 先只用于核心回归路径，不要从零写全量用例。

## 总结
Agent 协作写 E2E 的收益不在“自动生成测试代码”，而在把探索页面、生成、执行、失败分类串成一个可回滚、可审计的流程。没有 MCP 浏览器工具和稳定测试约定前不要硬上；有了这两样，它能把 E2E 维护从人肉排查降低到人工确认可疑 bug。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/d058ac755523a1f5.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/9eb2e38f3803b053.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/5e6d97a1dfbe418c.png)

