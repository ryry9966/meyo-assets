---
title: 自动化测试的 Agent 协作：让 AI 帮你写 E2E 测试
feedId: 35272
source: 综合讨论
publishedAt: 2026-08-30
---

## 背景

E2E 测试的维护成本通常比单测高：选择器变化、登录态失效、数据准备麻烦、失败日志难读。过去靠人肉补脚本，现在可以用 OpenClaw 这类 Agent 编排平台，把浏览器、测试运行器、代码仓库工具接成 MCP 服务，让模型参与“生成—运行—修脚本”的闭环。

好处不是“一键生成整个测试套件”，而是把高频、重复的调试动作交给 Agent，人负责审边界和改断言质量。

## 问题

直接让 AI 写 E2E 测试，常见问题是：

- 不动真实 DOM，凭经验猜选择器；
- 断言写得太死，比如依赖页面文案或动态时间；
- 不读失败日志，反复生成同样错误；
- 一次生成太多用例，上下文爆掉；
- 误改共享测试环境或登录态。

所以需要一个工程化流程，而不是一句 prompt 命令。

## 做法/步骤

我在 OpenClaw 中使用的组合是：Playwright MCP 负责浏览器操作与 DOM 读取，GitHub MCP 负责建分支、提交，本地 Docker 容器跑 Playwright test，输出 JUnit/JSON 结果。工具白名单只保留读取页面、运行测试、读日志、提交代码这几个动作。

第一步，写测试意图卡，不直接让 Agent 自由发挥：

```yaml
scope: checkout
user_path:
  - open /cart
  - click data-testid=checkout-btn
  - fill shipping form
assertions:
  - order summary visible
  - total > 0
forbidden:
  - do not edit production data
  - do not change seed users
```

第二步，让 Agent 先打开页面读 DOM，基于 `data-testid`、`role`、`text` 定位元素，不允许写死 CSS class。生成初版脚本后，先在隔离容器里跑一遍。

第三步，失败闭环。把失败步骤、DOM snapshot、console error、网络错误截断后回传给 Agent，并限制它只能修改对应 spec 文件。修复后再跑，直到通过或达到轮次上限。

第四步，通过后生成 PR，但默认不自动合入。人只看 diff、断言和测试范围。

## 踩坑点

- **选择器脆弱**：页面改版后 CSS class 经常变。优先让前端加 `data-testid`，或者用 role/text，别默认用结构化 CSS 路径。
- **状态污染**：登录态、购物车、优惠券必须由 fixture 或 seed 脚本创建。Agent 很容易直接复用上一次浏览器 session，导致用例“假通过”。
- **断言幻觉**：模型可能生成“页面一定包含某个文案”的错误断言。人工必须检查断言是否真正表达业务预期。
- **上下文爆炸**：不要一次回传完整 trace。截取失败步骤前后 20 行 DOM 和相关网络请求即可。
- **权限过大**：不要给 Agent 直接操作生产环境、支付、退款或用户数据的工具权限。测试环境也要做数据隔离。

## 可复用建议

把测试意图卡做成模板，放进团队知识库；每个业务模块维护一份 `specs/checkout.yaml` 这样的文件，Agent 只读取并实现，不自己发明测试范围。

将失败案例沉淀成“常见修复提示”，例如“登录态失效时先检查 storageState fixture”“超时不一定等 30 秒，先确认元素是否真的出现”。这些提示可以作为 system prompt 的补充，减少反复试错。

在 OpenClaw 里为测试 Agent 单独建一个 workspace，每次运行使用干净的快照或容器，避免残留文件影响判断。

## 总结

Agent 协作写 E2E 测试的核心不是替代测试工程师，而是把“打开浏览器、读错误日志、改选择器、再跑一次”这种重复闭环自动化。人保留两件事：定义业务路径和审查断言。控制好工具权限、上下文长度和测试环境隔离后，它能稳定地帮你把回归用例从“三天补完”压到“半天生成 + 一天审核”。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/26d16858be551623.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/dee887f719e5f3d9.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/40271c35b120f03e.png)

