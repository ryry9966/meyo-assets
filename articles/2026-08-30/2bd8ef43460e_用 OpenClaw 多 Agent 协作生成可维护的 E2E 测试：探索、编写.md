---
title: 用 OpenClaw 多 Agent 协作生成可维护的 E2E 测试：探索、编写、验证与修复
feedId: 35328
source: 综合讨论
publishedAt: 2026-08-30
---

## 背景

端到端测试（E2E）在很多项目里是最贵的一层：界面一改，选择器失效；手写用例重复度高，录制工具生成的代码又难维护。把“AI 帮你写 E2E”当成单次代码补全，通常不靠谱，因为真实页面里的交互路径、状态变化和断言设计需要一个闭环，而不是一次性生成。

## 问题

单 Agent 直接生成 E2E 测试，常见三类问题：

- 选择器脆弱：模型偏好 `.css-xxx` 或深层 XPath，界面轻微调整就挂。
- 断言太弱：只检查元素存在，不检查提交后的状态变化，容易产生 false positive。
- 没有真实执行：生成完就结束，代码看起来像样，跑起来失败一堆。

所以，我们需要把“探索页面 → 生成用例 → 执行验证 → 反馈修复”串成一个可重复的流程。

## 做法：用 OpenClaw 组织三个 Agent

核心不是一个神级 prompt，而是把流程拆成可回收的步骤。借助 OpenClaw 的多 Agent 协作，并接入 Playwright MCP 访问真实页面。

### 1. 接入 Playwright MCP

在 OpenClaw 中注册 Playwright MCP server，提供 `navigate`、`snapshot`、`click`、`type`、`assert` 等工具。建议跑在 Docker 或隔离的浏览器 profile 中，避免污染日常登录态。

### 2. 定义三个 Agent

- **Explorer**：打开目标页面，按指定业务路径操作，输出结构化交互记录。
- **TestWriter**：根据 Explorer 的步骤和用户故事生成 Playwright 测试。
- **TestRunner**：执行测试，收集失败信息，反馈给 TestWriter 修复。

### 3. 设置协作规则

Explorer 输出 JSON，字段包含 `element`、`action`、`selector_hint`、`expected_state`。`selector_hint` 必须优先 `data-testid`，其次 `getByRole` / `getByLabel`，禁止生成 `.css` 路径。TestWriter 只消费 JSON，不自行想象 DOM。TestRunner 要返回真实错误栈和截图路径，而不是“测试通过”。

### 4. 示例工作流

输入目标场景：“登录后创建项目并校验成功提示”。

1. Explorer 执行：进入 `/login`，填用户名密码，点击登录，打开 `/projects/new`，填写名称，提交，记录成功 toast 的状态。
2. TestWriter 生成 Playwright spec。
3. TestRunner 在干净环境跑一遍。
4. 若失败，TestRunner 把错误回传给 TestWriter，例如：`getByText('创建成功') 超时，实际页面还存在表单错误提示`。TestWriter 修正断言或选择器。
5. 循环最多 3 次。输出可提交的 spec 文件和运行报告。

## 踩坑点

- **选择器漂移**：必须在 Explorer 阶段约束 `selector_hint`，否则模型会生成深度 CSS。最好在测试基类里封装 `getByTestId` 和 `getByRole` 的短方法。
- **幻觉验证**：不要让 TestWriter 或 Explorer 说“测试通过”。通过状态只能来自 TestRunner 的实际 exit code 和报告。没有 Runner 输出，不能标记为可用。
- **DOM 上下文爆炸**：`snapshot` 全量 DOM 会耗尽上下文。Explorer 要使用可访问性树或过滤后的元素列表，只保留可交互节点。
- **断言弱化**：提示词中必须给出反例：“不要只检查元素存在，要检查提交后路由变化、成功提示文本、或网络响应状态”。否则容易生成 `expect(page.locator('button')).toBeVisible()`。
- **循环修复陷阱**：限制修复最多 3 轮，失败后保留失败日志和截图，要求人工介入。自动修复太多轮会生成扭曲的测试来“适应当前 bug”。

## 可复用建议

- 把 Explorer 的输出当作中间产物，纳入版本管理，方便追溯测试生成依据。
- 为不同业务页面维护一份 selector 规范，放进 prompt 模板，而不是每次现写。
- 使用 OpenClaw 的 memory 或文件系统工具保存项目级测试基类与 fixtures，TestWriter 生成代码时优先复用。
- 先拿 1-2 个稳定流程跑通闭环，再加入权限、边界状态等复杂场景。
- 对每次生成结果做 `git diff`，确认没有污染已有用例。

## 总结

Agent 协作的价值不在于自动写出完美的 E2E，而在于把探索、生成、执行、修复变成一条可重复、可审计的流水线。把“选择器规范、执行验证、失败反馈”固定下来之后，AI 产出才可能真正进入日常 CI。单次生成可能惊艳，只有闭环才能复用。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/f9569046d0b73a1d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/5680f124ef6447ac.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/9b80c1fac941c3f3.png)

