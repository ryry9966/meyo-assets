---
title: Agent 协作写 E2E 测试：一个可落地的探索-生成-验证闭环
feedId: 34336
source: 综合讨论
publishedAt: 2026-08-23
---

## 背景

E2E 测试长期维护成本高：手工编写慢，选择器脆弱，需求变更后用例容易失效。让 LLM 直接生成测试代码听起来很美好，但一次性生成的用例往往存在选择器错误、断言不合理、甚至无法运行的问题。我们在 OpenClaw 里尝试把 AI 从“单次生成器”变成“协作 Agent”，通过 MCP 工具打通浏览器操作和测试执行，让 AI 在闭环里迭代，效果比单次生成稳定得多。

## 问题

直接让 LLM 写测试有三个典型痛点：

1. 缺乏页面真实结构，容易编造选择器。
2. 无法执行验证，语法错误或运行时错误需要人工排查。
3. 测试失败后模型不会自己修复，只能重新生成，效率低。

这些问题本质上是“生成”和“验证”割裂。我们需要一个能操作浏览器、运行测试、读取失败信息并触发修复的协作流程。

## 做法 / 步骤

### 1. 准备 MCP 工具

我们使用两个 MCP：

- **Playwright MCP**（官方）：暴露 `browser_navigate`、`browser_click`、`browser_snapshot` 等能力，用于探索页面。
- **自定义 test-runner MCP**：封装 `pnpm playwright test <file>` 和读取 JSON 报告的接口，用于执行测试并返回失败信息。

在 OpenClaw 中注册这两个 MCP，并按角色分配工具权限。

### 2. 定义三个协作角色

- **Explorer**：使用 Playwright MCP 探索目标页面。输入为测试意图，输出为页面关键路径、可访问性快照和可用选择器列表。这里我们只允许使用 accessibility snapshot，不允许使用截图，以减少模型幻觉。
- **TestWriter**：根据 Explorer 的输出和测试意图生成 Playwright 测试代码。必须使用 `data-testid`，禁止使用 CSS 类名。输出到 `tests/` 目录。
- **Verifier**：调用 test-runner MCP 执行测试。如果失败，提取错误信息（例如超时、选择器未找到、断言不符），返回给 TestWriter 要求修正。

在 OpenClaw 中，这三个角色对应三个 task，每个 task 有明确输入输出，并设置最大迭代次数（通常 5 次）。

### 3. 闭环迭代

一次典型流程：

- 用户提供测试意图：“验证登录成功后跳转到 dashboard 并显示用户名”。
- Explorer 打开 `localhost:3000/login`，获取快照，记录 input、button 的无障碍信息。
- TestWriter 生成 `login.spec.ts`。
- Verifier 运行测试，发现错误：`expect(page).toHaveURL(/dashboard/)` 失败，实际跳转是 `/app`。将错误返回。
- TestWriter 修正断言。
- 再次运行通过。

这个闭环的关键是失败原因能被结构化地传回给生成方，而不是简单的“报错了”。

## 踩坑点

1. **选择器漂移**：模型倾向于使用 `.btn-primary` 这类易变类名。必须在 system prompt 中强制约束，并在 Verifier 中做代码检查，包含禁用模式则直接打回。
2. **模型幻觉**：Explorer 如果看截图，容易编造不存在的 DOM 节点。我们只提供 accessibility snapshot，不提供截图，幻觉明显减少。
3. **测试环境污染**：重复运行会累积数据，导致断言不稳定。需要在 test-runner 中加入全局 setup，每次运行前重置数据库和 localStorage。
4. **Context 膨胀**：页面快照可能很大，Explorer 输出需要截断，只保留交互元素和语义角色，否则 TestWriter 会丢失关键信息或超出 token 上限。
5. **无限修复循环**：必须设最大迭代次数，超过后标记为待人工处理，不要无限重试。否则可能陷入“修复 A 坏 B”的循环。

## 可复用建议

- **用 YAML 描述测试意图**，而不是自然语言。例如：

```yaml
intent:
  entry: /login
  steps:
    - fill: username
      value: test@example.com
    - fill: password
      value: secret123
    - click: submit
  expect:
    url: /app
    text: "Welcome"
```

这样各 Agent 的输入一致，减少歧义。

- **强制使用 `data-testid`**：前端开发规范里要求埋点，AI 只允许使用该属性，能显著降低选择器脆弱性。
- **保留人工 review gate**：AI 生成的测试进入 PR 前必须经过人工 review，避免错误断言被当作通过。
- **复用 Playwright codegen 记录**：把这些记录作为 few-shot 示例提供给 TestWriter，生成质量会更高。
- **记录每次迭代的失败原因和修复 diff**：方便分析模型常见错误，持续优化 prompt。

## 总结

Agent 协作写 E2E 测试的价值不在于“自动写出完美用例”，而在于把探索、生成、执行、修复变成一个可重复的工程闭环。用 MCP 暴露工具，用 OpenClaw 编排角色，配合强约束和人工审核，可以显著降低维护成本。当前阶段它更适合生成冒烟测试和回归用例，复杂业务场景仍需人工参与。不要指望它一上来就替代测试工程师，把它当成一个不知疲倦的结对伙伴，效果会比较实际。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/0191365a29e59c37.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/3045bc9607fd0cc0.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/e5e231425b633d3f.png)

