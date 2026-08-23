---
title: Agent 协作写 E2E：从“能生成”到“能维护”的工程化路径
feedId: 34389
source: 综合讨论
publishedAt: 2026-08-23
---

# 自动化测试的 Agent 协作：让 AI 帮你写 E2E 测试

## 背景

E2E 测试的长期成本从来不在“写第一条用例”，而在维护。选择器随着前端改版失效，业务路径变化后旧用例没人敢删，最后测试套件变成 CI 里的装饰品。Agent + MCP 的出现让浏览器操作可以被模型调用：读取可访问性树、点击、输入、截图、运行测试并读取失败信息。这不再是“让 ChatGPT 凭空生成测试代码”，而是让 Agent 进入真实的反馈循环。

如果你已经在 OpenClaw 里接过 Playwright MCP 或 chrome-devtools MCP，大概率见过它能自己打开页面、操作 DOM。但要让这件事稳定产出可用测试，需要一些约束。

## 问题

直接对 Agent 说“帮我给下单流程写 E2E 测试”，结果通常有几个通病：

- 选择器脆弱：AI 偏好长 CSS 路径或 XPath，如 `#app > div:nth-child(2) > span`，一改版就碎。
- 断言太弱：只检查某个文字出现，不验证 URL、状态或数据变化。
- 上下文爆炸：把整个 DOM 塞给模型，生成速度慢且容易跑偏。
- 误操作：在真实环境点击删除、支付、权限变更等危险按钮。
- 不可运行：生成代码缺少 import、等待策略错误、选择器不存在。

所以关键不是“让 AI 写”，而是给它一个可观察、受限、可回放的环境，并规定选择器和断言规则。

## 做法 / 步骤

### 1. 先让 Agent 做侦察，而不是直接写代码

接好 Playwright MCP 后，先不要下“帮我写测试”的指令。我会准备一个契约文件，例如：

```yaml
# specs/login.yaml
path: /login
steps:
  - fill "邮箱"
  - fill "密码"
  - click "登录"
assert:
  - url: /dashboard
  - heading: 工作台
selectors:
  prefer: ["data-testid", "role"]
  ban: ["css_long_path", "xpath"]
```

然后让 Agent 打开目标页面，读取可访问性快照，确认元素的 role、name、testid。这个过程的产出不是测试代码，而是对页面结构的确认：哪些元素有稳定 testid，哪些只能通过 role 定位。

### 2. 约束选择器优先级

必须明确告诉 Agent：优先 `data-testid`，其次 `getByRole` + name，最后才是稳定文本。禁止裸 CSS 和 XPath。示例：

```ts
await page.getByRole('button', { name: '登录' }).click();
await expect(page).toHaveURL(/\/dashboard/);
await expect(page.getByRole('heading', { name: '工作台' })).toBeVisible();
```

这个约束能显著降低后续维护成本。

### 3. 分旅程生成，不要一次性铺全量

一个 `spec` 文件只对应一个用户旅程。比如“访客登录”“会员加入购物车”“管理员导出报表”分开。每个旅程控制在 5-8 步以内。上下文越聚焦，生成质量越高。

### 4. 用失败输出驱动修复

生成代码后，在本地运行 Playwright。将失败输出、错误行号、相关 trace 回传给 Agent，要求它给出 diff，而不是重新生成整个文件。限制最多 3 轮修复。每次修复必须说明根因，例如“`getByText('登录')` 匹配到多个元素，改为 role + name”。

## 踩坑点

- **选择器脆弱**：AI 很爱长 CSS/XPath。必须在提示词里明确禁止，并在 review 时人工拦截。
- **自动执行误操作**：测试环境要隔离真实支付、权限、删除操作。对 Agent 的操作做白名单，例：只允许点击 `data-testid` 开头的元素，或使用 dry-run 模式。
- **上下文膨胀**：不要给完整 HTML。让 MCP 返回可访问性树或裁剪后的结构，只保留 main 区域。页面快照超过一定长度就要求 Agent 先定位再读取局部。
- **断言过弱**：只断言 `expect(page.getByText('成功')).toBeVisible()` 不够。要断言 URL 变化、状态码、数量变化、按钮 disabled 状态等。
- **死循环修复**：有时候 Agent 会反复在错误方向上改。限制修复轮次，并要求它先解释失败原因，再给 patch。
- **固定 sleep**：生成代码里容易出现 `await page.waitForTimeout(3000)`。要求使用 `expect` 自动等待，不要凭感觉等。

## 可复用建议

- 把登录、创建订单、清理数据这类操作封装成业务级 MCP 工具或 OpenClaw 技能。Agent 调用 `login_as` 而不是每次自己摸索表单。
- 建立 `specs/` 目录作为测试意图源。代码生成只是实现，意图文件长期可读。
- Agent 生成的测试必须过人工 review。CI 里可以做“Agent 辅助修复”，但默认不直接合入主分支。
- 保留失败 trace 和录屏。Agent 修复时不要只看报错，要让它读 trace 中的快照和网络请求。
- 把选择器约定、禁止项、断言要求沉淀成提示词模板，复用给所有测试任务。

## 总结

Agent 写 E2E 测试的真正瓶颈不在生成，而在约束和反馈。把浏览器能力通过 MCP 暴露给 Agent，用契约文件限制范围，用失败输出驱动修复，再叠加选择器规范与操作白名单，AI 就能从“偶尔能写出来”变成“可以维护地写”。

最终目标不是让 AI 替你写一次性脚本，而是让测试意图结构化、固定化，让 Agent 在稳定的工程护栏里持续产出。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/4c1817225e6337b9.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/2edf9955d7280e4b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/1007c2a53fbf62cc.png)

