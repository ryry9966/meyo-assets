---
title: 让 Agent 协作写 Playwright E2E 测试：从页面理解到自愈修复
feedId: 32030
source: 综合讨论
publishedAt: 2026-08-07
---

## 背景：E2E 测试的“最后一公里”难题

端到端（E2E）测试是保障用户流程完整性的关键，但维护成本极高。界面微调、DOM 结构变化、异步加载时机变化，都会让用例报错。手写用例时，我们花大量时间调试选择器、等待逻辑和断言，却仍会遗漏边界情况。

大语言模型可以凭自然语言描述生成测试代码，然而直接让一个 Agent 生成本地能跑的 Playwright 脚本通常不可靠：选择器不稳定、缺乏对当前页面的真实感知、无法处理动态组件。真正可落地的方案，需要将“理解页面”、“设计用例”、“编码”、“验证”与“修复”拆分为多个协作的 Agent，各自利用合适的工具，形成闭环。

OpenClaw 的 Agent 协作能力与 MCP 协议的结合，恰好能够搭建这样一套工作流。下面按照工程实践的顺序，给出一个可复现的步骤。

## 问题拆解：为什么单一 Agent 不够

如果你曾用 ChatGPT 或某个 Agent 直接生成 `page.click('.submit-btn')`，大概率会收到 `selector resolved to hidden` 或超时错误。原因有三：

1. **缺乏页面上下文**：Agent 不知道当前页面的真实 DOM、交互元素和组件状态。
2. **选择器策略单一**：仅凭 CSS 类名或文本，没有使用 `data-testid`、`aria-label` 等多层降级策略。
3. **无执行反馈**：生成即结束，失败后需要人工介入修改。

因此，我们需要的不是一个“写测试的 Agent”，而是一个多角色协作的测试工厂。

## 实践步骤：用 OpenClaw 搭建多 Agent 测试生成流水线

### 1. 整体架构

四个核心 Agent，通过 OpenClaw 的任务编排协同工作：

- **Explorer Agent**：挂载 Playwright MCP server，访问目标页面，提取 DOM 快照、可交互元素列表及其属性（包括自定义的 data 属性、可见文本、空白占位等），输出结构化 JSON。
- **Test Designer Agent**：接收用户故事或测试场景描述（如“登录后创建订单并验证状态流转”），输出测试计划 YAML，包含步骤序号、动作描述和预期结果。
- **Coder Agent**：结合 Explorer 输出的页面描述和 Designer 的测试计划，生成 Playwright 测试脚本（TypeScript）。要求使用多层选择器策略：优先 `getByTestId`，其次 `getByRole`/`getByLabel`，最后才使用稳定的 CSS 选择器。
- **Executor & Validator Agent**：实际运行生成的测试，捕获失败信息（截图、日志、DOM 状态）。若失败，判定是脚本逻辑错误还是页面缺陷，若是前者，将错误上下文与原有提示词一起交给 Coder Agent 修复，形成最多 3 轮的自动修复循环。

这些 Agent 之间的通信全部通过自然语言+结构化数据完成，OpenClaw 的任务队列负责编排和状态持久化。

### 2. 关键实现细节

#### Explorer Agent：如何提取有效页面描述

利用 Playwright MCP server 的 `browser_snapshot` 类工具，可以获得当前页面的可访问性树（包含 role、name、level 等）。但直接 dump 会包含大量无关节点，需要经过二次处理：

- 只保留可交互元素（button、link、textbox、combobox 等）。
- 提取其可见文本（innerText 截断至 50 字符）。
- 如果存在 `data-testid`、`data-cy`、`aria-label`，一并保留。
- 对 form 元素，记录 placeholder 和关联的 label 文本。

输出格式约定为 JSON，例如：
```json
{
  "url": "/dashboard",
  "elements": [
    {
      "role": "button",
      "name": "新建订单",
      "testid": "create-order-btn",
      "ariaLabel": null,
      "visible": true
    }
  ]
}
```
这样 Coder Agent 可以立即确定可用的选择器。

#### Test Designer Agent：测试计划的标准化

测试计划用 YAML 定义，清晰且方便后续解析：
```yaml
test_name: "用户下单后查看物流轨迹"
steps:
  - action: "点击「新建订单」按钮"
    expected: "跳转到订单创建页面"
  - action: "填写收货地址并提交"
    expected: "进入订单详情，状态为「待发货」"
  - action: "模拟物流更新后点击「查看轨迹」"
    expected: "弹窗展示轨迹地图"
```
这个 YAML 作为 Coder Agent 的主要行为指导，也作为后续断言的基础。Designer Agent 不需要知道具体页面元素，仅根据业务语义生成。

#### Coder Agent 与自愈修复

Coder Agent 的提示词包含：

- 页面元素描述 JSON
- 测试计划 YAML
- 编码规范：使用 `test.describe` / `test`，每个步骤独立成行，优先使用 `page.getByTestId()`，并加入显式等待 `waitForSelector` 或 `waitForResponse`。
- 额外工具：允许 Coder 请求重新获取某一页的 Explorer 输出（当发现元素信息不足时）。

脚本生成后，Executor 直接运行。如果失败，错误信息（简化后的 stack trace、失败时的页面截图描述）连同原始 YAML、原始 JSON 一并返回 Coder 进行修正。修正时，Coder 会调整选择器或添加等待条件。限制最多修正 3 次，避免死循环。

### 3. 踩坑点

- **Playwright MCP 连接断连**：长时间运行可能因 WebSocket 超时断开。解决方案是在 OpenClaw 的任务配置中对 MCP 调用进行重试和超时控制，并让 Explorer 在每次任务开始时检查连接状态。
- **动态内容导致 Explorer 快照过时**：单页应用中，页面经过点击后 DOM 改变，但 Explorer 是在初始快照基础上工作的。解决方法：Coder Agent 生成的测试中，每到一个新页面，都会调用一个自定义工具让 Explorer 重新抓取当前页面，然后把新描述注入上下文。这增加了 token 消耗，但极大提高了准确性。
- **选择器依然脆弱**：即使有 structured data，Coder 有时仍会写出 `.item-list > div:nth-child(2)`。在提示词中强约束+反面示例能改善，但不能完全杜绝。可以在校验阶段扫描生成的代码，对高风险选择器发出警告，必要时再次请求修正。
- **Validator 难以区分前后端错误**：例如一个“点击提交后页面无反应”的失败，可能是测试等待超时，也可能是后端返回 500。通过分析截图和 Playwright trace 有助于判断。我们的做法是让 Validator 检查网络响应状态码，若多次出现 5xx，则标记为环境问题而非测试脚本问题，停止重试并通知人工。

## 可复用建议

- **为测试生成建立合约**：页面元素需要稳定的 testid，这是整个流水线最核心的依赖。推动前端在开发时加上 `data-testid`，回报远超成本。
- **缓存与增量更新**：Explorer 的页面描述可以按 URL+版本号缓存，避免每次测试生成都重新抓取。OpenClaw 中可以借助 memory 或外部键值存储实现。
- **模板化测试计划**：将常见用户流程（登录、注册、下单、搜索）做成模板，Test Designer 实际填入的是业务数据，这样 Coder 生成的代码更稳定。
- **监控失败模式**：每次修复循环结束后，记录最初的问题选择器类型和最终成功的选择器类型。长期可以训练一个专用于选择器修正的小模型或规则引擎，进一步减少 LLM 调用次数。

## 总结

让 Agent 帮你写 E2E 测试，并不是直接丢一句话就完事。将它拆分为理解页面、设计用例、编码验证和修复的协作小团队，才能真正适应界面的多变。OpenClaw + Playwright MCP 的方案，将测试生成从一次性脚本输出变成持续维护的工程管道。当团队能接受“测试也由 Agent 维护”的理念时，回归测试的编写门槛会大幅降低，更多精力可以回到业务逻辑本身。

这或许就是 Agent 时代软件测试的新常态：人类描述意图，机器负责落地与修复。

---

