---
title: 让 Agent 协作写 E2E：一条 OpenClaw 上的可落地流水线
feedId: 34166
source: 综合讨论
publishedAt: 2026-08-22
---

## 背景

E2E 测试有价值，但维护成本很高。常见痛点：选择器脆弱、页面一改用例就失效、新增页面补测试太慢。引入 Agent 后，很多人会直接说“帮我写一个登录测试”，结果生成的代码能跑通 demo，却很难真正进入仓库——选择器随意、断言过浅、缺少等待策略和可维护结构。

问题不是 AI 不能写，而是缺少工程化边界。单个 Agent 既探索页面又写断言，很容易把“页面现状”当成“业务正确性”。

## 做法：拆成两个 Agent + 一个 MCP 工具面

在 OpenClaw 里，我拆成三个角色：

- **Explorer**：通过 Playwright MCP 打开页面、读取 DOM/可访问性树/截图，输出页面模型和关键流程，不负责写最终测试。
- **TestWriter**：读取 Explorer 输出的页面模型、现有测试模板和业务规则，生成或修改 spec。
- **Runner**：本地执行测试，收集失败日志、截图和 DOM 快照，作为回归输入。

工具面通过 MCP 暴露浏览器和仓库操作，OpenClaw 的任务编排控制它们之间的交接。

### 具体步骤

1. 给 Explorer 一个页面 URL 和“只探索，不修改断言”的指令，让它输出：
   - `page.md`：页面区域、关键元素、可访问性名称、data-testid 或稳定选择器建议。
   - `flows.json`：从用户视角描述关键操作序列，比如“登录 → 打开订单列表 → 筛选待支付”。

2. TestWriter 读取 `page.md`、`flows.json` 和仓库里的 `e2e/templates/`，生成 Playwright spec。要求：
   - 优先使用 `getByRole` / `getByLabel` / `data-testid`。
   - 等待策略使用断言而非固定 `sleep`。
   - 每条用例包含一个业务断言，不能只断言“页面不报错”。

3. Runner 跑测试。失败后把日志、截图、DOM 快照回传给 TestWriter。TestWriter 只允许修订定位逻辑，不允许静默弱化业务断言。若断言需要调整，必须产出 `NEED_REVIEW` 标记给人工。

4. 人工只 review diff 和 `NEED_REVIEW`，不从头手写测试。

## 踩坑点

- **选择器漂移**：Agent 第一次给出的 CSS 路径通常很脆弱。强制它使用可访问性树，而不是复制浏览器 DevTools 的 XPath。没有 data-testid 的页面，先补测试属性，再让 Agent 生成。

- **自动修复把断言改松**：这是最常见的坑。失败后 TestWriter 会把“期望出现订单列表”改成“期望页面加载完成”，测试绿了但没意义。必须把业务断言设成不可自动降级。

- **复杂 DOM 结构**：iframe、shadow DOM、虚拟滚动列表里元素未渲染，Explorer 需要滚动或展开操作，否则会误报元素不存在。给 Explorer 增加“操作前先滚动到可交互区域”的指令。

- **上下文膨胀**：一次会话塞入太多页面，Explorer 后期开始遗漏步骤。我限制每个任务最多探索 3 个页面，并让中间产物落盘，而不是全部放进上下文。

- **CI 环境差异**：本地能跑，CI 超时。固定 viewport、baseURL、网络 mock 策略，给 Runner 提供独立的 MCP 配置，避免依赖本地插件状态。

## 可复用建议

- **把“探索”和“断言”隔离**：探索可以自动，断言变更要人工确认。
- **建立 golden flows 库**：把关键业务路径的 `flows.json` 版本化，作为回归基线。
- **让页面模型成为唯一事实源**：`page.md` 生成后提交到仓库，后续用例只引用它，不让每个测试单独写选择器。
- **模板优先**：先做 3-5 个高质量 spec 作为 few-shot，比写很长的系统提示更稳定。
- **小步跑**：先覆盖一条冒烟路径，跑通整个协作流，再扩展到更多页面。

## 总结

Agent 写 E2E 的价值不在“全自动生成”，而在于把繁琐的页面探索和初稿工作分出去。工程化地拆角色、落中间产物、限制断言修改权限后，生成的测试才可能进入仓库长期维护。OpenClaw 的 Agent 协作 + MCP 浏览器工具面，适合搭这样一条小流水线：Explorer 看页面，TestWriter 写代码，Runner 跑结果，人类守边界。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/3c3971d5fd41eac4.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/9962590adc3d808c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/dee836a8e0814f75.png)

