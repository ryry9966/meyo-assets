---
title: 让 Agent 写 E2E：基于 Playwright MCP 的协作式测试生成与修复
feedId: 34172
source: 综合讨论
publishedAt: 2026-08-22
---

## 背景

E2E 测试最大的成本通常不是首次编写，而是长期维护。页面一改，选择器失效；交互一多，等待策略失控；换一个环境，登录态和 baseURL 又出问题。

AI 确实能生成测试代码，但如果直接把任务丢给 Agent，得到的往往是“能跑通但不可维护”的脚本，甚至在真实项目里根本跑不起来。更合理的方式，是把 Agent 当成一个需要强约束的协作者：它负责探索页面、生成候选脚本、执行并修复，人负责定规则、做 review 和控架构。

在 OpenClaw 这类 Agent 框架里，MCP 可以承担浏览器能力边界，prompt 则是协作协议。

## 问题

直接让 Agent 写 E2E 时，常见痛点包括：

- 选择器脆弱，喜欢用深 CSS 层级或动态 class
- 对 iframe、shadow DOM、动态渲染缺少判断
- 测试里大量插入 `waitForTimeout`
- 登录步骤被重复写在每个测试里
- baseURL 硬编码，本地和 CI 不一致
- 只断言“元素存在”，不验证真实业务结果

所以核心不是“能不能生成”，而是生成后能不能进入工程维护。

## 做法 / 步骤

### 1. 通过 Playwright MCP 把浏览器能力交给 Agent

配置一个 Playwright MCP Server，让 Agent 具备真实浏览器的 `navigate`、`snapshot`、`click`、`fill`、`screenshot` 等工具。

```json
{
  "mcpServers": {
    "playwright": {
      "command": "npx",
      "args": ["@playwright/mcp@latest"]
    }
  }
}
```

启动后，Agent 不再只能凭经验猜 DOM，而是可以看到可访问性树和页面快照。

### 2. 使用严格的任务模板，而不是自由发挥

给 Agent 的指令要具体到选择器策略和等待策略。例如：

```text
请为以下用户路径编写 E2E 测试：
- 起始 URL: /login
- 用户步骤: 登录 -> 进入仪表盘 -> 新建项目
- 断言: 出现“项目创建成功”，且左侧导航高亮“项目”

要求：
1. 优先使用 data-testid，其次 role/text。
2. 禁止使用纯 CSS 深度层级和动态 class。
3. 等待使用 expect(...).toBeVisible({ timeout: 5000 })，禁止 waitForTimeout。
4. 登录态封装到 setup 或 storageState，测试文件不要重复 UI 登录。
5. baseURL 必须使用 process.env.BASE_URL。
```

先让 Agent 调用 `snapshot` 探索页面，再给候选 locator，而不是直接生成文件。

### 3. 执行失败后必须基于证据修复

生成测试后，进入“运行—观察—修复”循环。每次失败要提供：

- 失败步骤
- 失败截图
- 页面 snapshot 片段
- console error

让 Agent 根据这些证据修改，而不是盲目重写。这样可以减少“改了 A 步骤，B 步骤又坏”的情况。

### 4. 抽公共步骤，保留业务断言

要求 Agent 把登录、导航等重复步骤抽到 fixtures 或 helper 中。测试主体只保留核心业务路径和用户可感知的断言，例如文案变化、数据变化、跳转后的页面状态。

## 踩坑点

1. **MCP 工具暴露过多**  
   如果 Agent 可以随意调用所有浏览器工具，它会频繁导航、重复 snapshot，导致上下文爆炸。建议按任务裁剪工具集，只保留必要的几个。

2. **iframe / shadow DOM 会被误判**  
   可访问性树里有时看不到 iframe 内部元素，Agent 容易直接判定“元素不存在”。需要在 prompt 里要求确认是否存在 iframe，并显式切换 frame。

3. **登录态不能每次走 UI**  
   每次 UI 登录慢且不稳定。更实际的做法是用 API 登录后保存 `storageState`，或者通过全局 setup 种 cookie。

4. **`waitForTimeout` 掩盖 flaky**  
   Agent 为了“看起来稳定”，会插入固定等待。这会让测试短期通过，长期不可维护。prompt 里禁止，并在 CI 里做静态检查。

5. **环境不一致**  
   硬编码 `localhost:3000` 是常见问题。必须要求所有导航都基于 `process.env.BASE_URL`。

6. **断言太弱**  
   只检查元素存在，不检查业务结果，测试过了也没意义。至少保留一个用户可感知的断言。

## 可复用建议

- 把 Playwright MCP 包一层项目级 MCP，只暴露业务动作，例如 `login`、`navigate_to_dashboard`、`create_project`，降低 Agent 的自由度。
- 将 prompt 模板存入仓库，版本化共享，团队可以持续优化规则。
- 在 CI 中加入测试文件质量门禁，检查 `waitForTimeout`、深层 CSS 选择器、硬编码 URL 等坏味道。
- 保留生成过程和失败截图，作为人工 review 的证据。
- 把 Agent 当“测试实习生”：探索和修复可以交给它，但测试架构、选择器策略、断言标准必须由人定。

## 总结

Agent 协作写 E2E 的关键不是“一键生成”，而是把浏览器运行时能力通过 MCP 交给 Agent，同时用严格的任务模板、执行反馈和 review 约束它。

在 OpenClaw 场景下，MCP 是能力边界，prompt 是协作协议，CI 是质量底线。只有把这三层做好，AI 生成的测试才可能从演示脚本变成工程可用的资产。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/e1ef04faa7899bae.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/0301c4ccb3f26988.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/f7231badd78e2e4a.png)

