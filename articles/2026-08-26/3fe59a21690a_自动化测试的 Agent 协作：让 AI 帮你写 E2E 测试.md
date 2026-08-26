---
title: 自动化测试的 Agent 协作：让 AI 帮你写 E2E 测试
feedId: 34830
source: 综合讨论
publishedAt: 2026-08-26
---

## 背景

E2E 测试长期处于“最该自动化，但最难维护”的位置。选择器脆弱、页面改版、环境不稳定，让脚本很容易从资产变成负债。AI 生成代码看似能降低门槛，但单个 Agent 从零写 E2E，通常只能产出“看起来对的脚本”：选择器靠猜，断言不验证业务，遇到失败就加 `sleep`。问题不是 AI 不会写，而是它缺少工程上下文和分工。

OpenClaw 这类 Agent 编排工具适合解决这个问题。尤其是通过 MCP 把浏览器、Playwright、日志和测试运行器暴露给不同 Agent，让测试生成不再是一次性 Prompt 输出，而是一条可复用的流水线。

## 问题

如果只让一个 Agent 读需求直接写 Playwright 脚本，通常会遇到三类问题：

1. 选择器幻觉：AI 偏好 `text=`、`nth-child`、深层 CSS 路径，当前环境可能通过，换数据或换环境就挂。
2. 断言失真：AI 会把用户可见结果简化为“元素存在”，甚至为了通过而把断言改弱。
3. 修复不可控：失败后自动修复可能只是堆 `waitForTimeout`，没有真正定位原因。

所以更务实的做法不是追求“一键生成”，而是把 E2E 测试拆成多个 Agent 协作的流水线。

## 做法/步骤

### 1. 先绑定可观测能力，再让 Agent 动手

通过 MCP 暴露 Playwright 或浏览器工具，至少包括：导航、点击、输入、截图、DOM 快照、可访问性树、console/network 日志。给 Agent 提供只读查询能力，例如 `getByRole`、`locator.count()`、`page.url()`。

不要让 Agent 在信息不足时凭空写脚本。页面事实必须先于代码生成。

### 2. 拆成四个角色

- **Explorer**：探索页面路径，记录关键元素、可访问性名称、表单结构，输出 `route/action` 描述。
- **Spec Writer**：基于产品需求加 Explorer 的页面事实，生成测试草稿。要求使用 `data-testid`、`role`、`label`，禁止裸 CSS 或 `nth-child`。
- **Runner/Repairer**：执行测试，收集失败详情，最多修复三轮。修复必须基于失败时的 DOM 快照、console 和 network，不允许只加 `sleep`。
- **Reviewer**：检查断言是否验证用户可见结果，是否存在固定等待、脆弱选择器、重复用例。

可以用 OpenClaw 将这些角色串成任务流，每个 Agent 只读写自己的文件：

```yaml
agents:
  explorer:
    tools: [mcp__playwright__navigate, mcp__playwright__snapshot, mcp__playwright__query]
    output: exploration/page-checkout.json
  spec_writer:
    tools: [read_file, write_file]
    input: [exploration/page-checkout.json, requirements/checkout.md]
    output: specs/checkout.spec.ts
  runner:
    tools: [run_test, read_file, mcp__playwright__snapshot, mcp__playwright__console]
    output: runs/latest/failure.json
    max_repairs: 3
  reviewer:
    tools: [read_file, grep]
    checks: ["no_sleep", "no_nth_child", "assertion_visible_text", "unique_selector"]
```

这个 YAML 是示意，重点不是平台，而是把每个步骤的输入输出固定下来。

### 3. 用文件作为协作接口

`exploration/`、`specs/`、`runs/` 分别存放探索结果、测试草稿和运行现场。失败时把截图、DOM 快照、console error、network failed request、spec 文件打包成一个上下文，交给修复 Agent。这样每个 Agent 只依赖明确的输入，不会互相污染。

## 踩坑点

**AI 的“自信选择器”是最大坑。** 它经常输出 `text=Delete` 或 `div:nth-child(3)`，当前页面可能唯一，换环境就挂。必须在 Spec Writer 之后强制 Reviewer 检查，或者让 Explorer 返回可访问性名称和测试标识。

**自动修复容易把断言改弱。** Runner 为了通过，可能把 `expect(price).toBe('99.00')` 改成 `expect(price).not.toBeNull()`。必须锁定断言来源：需求里的期望值或 API 契约，不允许 Agent 自行删改。

**DOM 全量塞入会爆上下文。** 只给可访问性树或当前交互区域的 HTML 片段。长列表要截断，并标记元素数量。

**环境漂移。** Agent 运行时的端口、登录态、测试数据与人类调试时不一致。用固定 fixture 和 baseURL，失败时保存 trace、screenshot、storageState。

**权限控制。** 不要让 Agent 具有修改 CI 配置、跳过用例、直接合并代码的权限。写文件只在 `tests/` 草稿目录，合并走人工 PR。

## 可复用建议

- **建立页面对象，而非裸脚本。** 让 Spec Writer 生成语义化调用，同时维护 `pageObjects/` 目录，后续维护成本低。
- **每个测试场景附上验收描述：** 用户做了什么、看到什么。没有验收描述就不生成测试。
- **把失败现场打包成标准上下文。** 修复 Agent 只基于这个包工作，避免重新探索整个站点。
- **保留人工 review gate。** AI 适合产出草稿和修复建议，但合并前至少人工确认一次 pass rate、断言覆盖和 diff。
- **用小型回归集验证生成质量。** 每次生成后，故意注入一个已知失败，看 Agent 是否能定位并修复，而不是改写断言。

## 总结

Agent 写 E2E 真正有价值的形态不是“一键生成测试”，而是把探索、编写、执行、修复、审查拆开，让 AI 在受控上下文里协作。上下文比 Prompt 更重要，约束比模型更重要。OpenClaw 社区可以把这套流程落到 MCP 工具和文件接口上，得到一套可复用、可回滚、可审查的测试生成流水线。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/840359419ddbe31f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/55cd01a77cc3155f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/cf1133f027f6ec2b.png)

