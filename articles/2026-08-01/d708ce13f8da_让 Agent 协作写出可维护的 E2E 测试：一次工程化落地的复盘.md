---
title: 让 Agent 协作写出可维护的 E2E 测试：一次工程化落地的复盘
feedId: 31165
source: 综合讨论
publishedAt: 2026-08-01
---

# 让 Agent 协作写出可维护的 E2E 测试：一次工程化落地的复盘

## 背景：谁在为 E2E 测试买单？

端到端测试的维护成本人尽皆知——UI 一改，定位器失效；异步交互一加，sleep 到处飞；数据依赖一变，fixture 崩塌。即便如此，团队依然需要把核心用户路径签进 CI。我们日常用 Playwright 守护 3 条关键流程，但每两周的迭代还是让测试文件膨胀到难以维护的状态。

社区热议的“AI 写测试”大多停留在 demo 级别：给一张截图生成一段代码，或者让 LLM 根据自然语言描述输出脚本。这些方案在复杂业务场景下很快暴露三个问题：

- **定位策略粗暴**：喜欢用文本、placeholder、nth-child，缺乏工程化考量
- **缺乏状态意识**：生成脚本没有设置、断言、清理三部曲
- **可复用性为零**：每个用例都是“平铺直叙”，没有提取 page object 或 action 抽象

正因如此，我们尝试引入 **OpenClaw Agent 协作模式**：并不是让一个 Agent 独揽测试生成，而是让多个轻量 Agent 分工——探索、编码、审查——最终产出可直接提交到仓库的 spec 文件。

## 核心思路：把测试生成拆成 Agent 协作流水线

我们设计了三个 Agent 角色，运行在 OpenClaw 编排层，并通过 MCP 工具连接到 Playwright 浏览器和本地文件系统：

1. **Explorer Agent**（浏览器中探索）  
   接收“用户流程描述”（例如：登录 -> 新建项目 -> 邀请成员 -> 验证权限），在无头浏览器中逐步操作，记录每一步的：
   - 最终到达的 URL
   - 交互元素的可信选择器（优先 data-testid，其次 aria-label，再次 role-based）
   - 需要等待的网络请求或 DOM 状态（比如列表中至少出现一行数据）

2. **Scripter Agent**（在上下文中生成测试）  
   基于 Explorer 的录制轨迹和页面快照，使用内置的 Playwright 知识生成 TypeScript 测试代码，强制遵循我们定制的规则：
   - 使用 fixtures 中的 `testWithUser` 注入登录态，不硬编码凭证
   - 所有定位逻辑收敛到 Page Object，代码引用的选择器必须来自 Explorer 推荐的属性
   - 显式断言包含业务语义（不是 `expect(page.locator(...)).toBeVisible()`，而是 `expect(projectPage.memberList).toContainMember(email)`）

3. **Reviewer Agent**（在 CI 视角审查）  
   收到生成的测试文件后，模拟 Git diff 上下文，检查：
   - 是否引入了不稳定的等待（`page.waitForTimeout`）
   - 是否存在未封装的重复定位
   - 断言是否覆盖了关键后置状态
   - 生成反馈评论，Scripter 根据反馈迭代修改，直到通过审查或达到轮次上限

这三个 Agent 通过 OpenClaw 的任务队列串联，最终产出一个开好 Pull Request 的分支。人类只需要 review 最终结果。

## 落地步骤与关键决策

### 1. 为 AI 可读性设计选择器策略
我们在前端约定了一套**测试属性规范**：所有交互元素必须挂载 `data-testid`，且命名遵循 `模块-动作-对象`（如 `project-create-button`）。这一约定极大降低了 Explorer 的选择器猜测成本，也让 Reviewer 能轻松检查定位是否使用合规属性。

### 2. 让 Explorer 产出结构化轨迹而不是脚本
最初我们让 Explorer 直接生成 Playwright 代码，结果充满了 `page.click('text=Submit')` 的坏味道。后来改为产出 JSON 格式的“操作序列”，包含元素描述、推荐选择器、所需等待条件。Scripter 再根据这份描述生成代码，相当于在探索与编码之间做了解耦。

### 3. 内置 Playwright 最佳实践作为 Prompt 约束
我们在 Scripter 的系统 Prompt 中硬编码了 10 条规则，例如：
- 不使用 `page.waitForTimeout`，必须用 `waitForResponse` 或 `waitForSelector` 基于状态
- 所有 test 必须使用 `test.describe` 包裹，并配有 `test.beforeEach` 进行数据 setup
- 敏感操作（删除、权限变更）必须在断言后显式调用 cleanup

这些规则来自我们过去一年踩过的坑，Agent 生成代码时不能违反。

### 4. 用 MCP 绑定真实文件操作
通过 OpenClaw 的 MCP 文件系统接口，Scripter 可以将测试文件写入选定目录，Reviewer 可以读取 diff。所有修改都被限制在 `e2e/generated/` 路径下，避免误伤手写测试。

## 踩坑清单

- **异步渲染导致 Explorer 抓取不到元素**：即使加上 `waitForLoadState('networkidle')`，某些虚拟列表、骨架屏仍会干扰。最终我们在 Explorer 操作前注入 `page.waitForFunction` 检查特定的 DOM 标记（比如 `[data-loaded="true"]`）才继续。
- **Reviewer 的“洁癖”走到反面**：有一次 Reviewer 强制要求把所有选择器替换成 `data-testid`，但名单加载的列表项确实没有 testid，导致生成陷入死循环。解决方法是为 Reviewer 增加了豁免规则：当元素属于动态列表且无 testid 时，允许使用 `data-index` 结合 role 定位，并添加注释标记。
- **登录态复用容易造成测试间污染**：Scripter 初始倾向于在 `beforeEach` 中实时登录，导致执行缓慢。改成复用 storage state 后，需要让 Explorer 明确记录当前用户身份，避免多个用例串号。

## 可复用建议

1. **先建立团队的测试属性规范再做自动化** — 没有 data-testid 的页面，Agent 只能靠猜测，产出质量断崖式下降。
2. **从一条核心 Happy Path 开始** — 先用 Agent 生成一条最简单的流程，观察它在前端响应时间、加载模式下的表现，积累 Prompt 中的等待策略。
3. **别让 Agent 黑盒生成全部 API mock** — 让 Explorer 记录真实网络交互，再让 Scripter 基于 HAR 或拦截规则生成 mock，保持数据和 UI 的一致性。
4. **保留人类的一张“否决票”** — 我们的流程中，生成的 PR 必须经过人工 Approve，Agent 只负责消解 80% 的重复工作。遇到 Reviewer 和 Scripter 僵持，人工介入定规则，而不让循环无限进行。
5. **将成功的 Agent 输出反向注入 Prompt 示例** — 每当我们接纳一个生成良好的测试文件，就提炼其中的模式（比如 data cleanup 写法、table 断言方式）补充到 Scripter 的 few-shot 样例中，让后续生成更贴近团队风格。

## 总结

AI 写 E2E 测试的瓶颈从来不是“能不能生成代码”，而是“生成的代码能不能活过三个月”。通过 Agent 分工协作、工程化约束和审查闭环，我们把 E2E 生成从玩具级推向了可持续交付的水平。目前这套流水线覆盖了我们 60% 的回归用例生成，剩下的依然靠手写——但至少，我们不再害怕 UI 重构了。

如果你也在用 OpenClaw 做类似的 Agent 协作，欢迎带上你的 selector 策略和 reviewer prompt 一起交流。工具是现成的，但让测试活下来的，永远是那些刻进 prompt 里的工程规范。

---

