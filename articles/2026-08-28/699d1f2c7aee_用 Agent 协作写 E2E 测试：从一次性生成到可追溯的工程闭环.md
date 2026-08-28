---
title: 用 Agent 协作写 E2E 测试：从一次性生成到可追溯的工程闭环
feedId: 35040
source: 综合讨论
publishedAt: 2026-08-28
---

## 背景

E2E 测试是验证关键用户路径的最后一道防线，但维护成本高、选择器脆弱、断言容易漂移。很多团队的真实状态是：功能已经上线，E2E 还在补；或者测试挂了没人敢动，直接 skip。

引入 Agent 之后，很多人的第一反应是让模型直接读页面、生成 Playwright 脚本。实际跑下来，效果很不稳定：选择器可能不存在，断言过宽或过窄，第一次能跑通，第二次就断。

问题不在于“AI 不能写测试”，而在于缺少工程化协作。单个 Agent 一次性生成测试，就像让实习生凭截图交作业：没有执行反馈、没有评审、没有约束。

## 做法：拆成多个 Agent + MCP 工具

在 OpenClaw 这类 Agent 编排环境里，我会把 E2E 测试拆成 5 步。

**1. 上下文采集 Agent**

不要一上来就生成代码。先通过 MCP browser/DevTools 工具打开页面，抓取 accessibility tree、DOM snapshot、关键路由、按钮角色和名称，整理成结构化页面描述。

相比整页 HTML，ARIA/role 信息更接近用户行为，后面生成的 locator 也更稳定。

**2. 用例意图层**

让 Agent 先输出测试意图，而不是最终脚本。例如：

```json
{
  "action": "click",
  "target": {"role": "button", "name": "/提交/"},
  "assert": {"url": "/success|thank-you/"}
}
```

这一步的意义是：先确认业务路径正确，再关心实现。意图层也方便以后替换 runner。

**3. 生成与执行 Agent**

用 Playwright MCP 或 runner 工具把意图转成代码，并在干净上下文里执行。失败时截取 screenshot、console、network 信息回传，不让 Agent 盲目重试。

**4. 修复分类**

执行 Agent 必须区分“测试问题”和“产品缺陷”。

- 选择器失效、等待不足、断言过严，允许修复；
- 按钮真的不存在、接口返回错误、文案与预期不符，不允许修改断言掩盖，应该生成失败报告给人看。

**5. 人工 gate**

所有 Agent 生成的测试最终以 diff/PR 形式合入。Reviewer 只看三点：是否覆盖关键路径、是否引入脆弱选择器、是否为了通过而弱化断言。

## 踩坑点

**别把整页 HTML 喂给模型。** 太长，既费 token 又引入噪声。优先给 ARIA 树和必要的 DOM 片段。

**禁止 sleep 和脆弱 XPath。** Agent 默认喜欢用 `text=` 和 `nth=`，这些选择器易碎。要求优先 role、label、data-testid，并在工具层做 lint。

**无约束修复会掩盖 bug。** 必须给修复 Agent 设置规则，例如只能改 locator/wait/assertion scope，不能删除断言。

**浏览器上下文污染。** 复用 session 会导致状态残留、用例相互影响。每个用例跑在独立 context，必要时关闭缓存和登录态。

**生成-修复死循环。** 设置最大修复轮数，比如 2 次；仍失败就转人工工单，不要让 Agent 无限折腾。

## 可复用建议

- 把“测试意图”做成中间表示。以后无论底层换 Playwright、Cypress 还是其他 runner，意图层不用改。
- 让 Agent 维护 page object/fixture，而不是在用例里写死选择器。
- 只生成失败或缺失的测试，不要全量重写。全量重写会引入大量不确定 diff。
- 建立可回归的评测集：选 5-10 条历史线上问题路径，跑 Agent 生成的测试，看能否稳定复现和回归。
- 把日志、trace、截图都留在 artifact 里。Agent 的修复建议要能追溯到证据，而不是“可能这样改”。

## 总结

Agent 协作写 E2E 测试的收益，不在于自动产出大量脚本，而在于把“生成-执行-反馈-修复-评审”串成一个可追溯的工程闭环。

控制好输入噪声、限制修复权限、保留人工 gate，它更适合做补测试和修脆弱用例，而不是替代测试设计。真正能落地的方案，通常是少量 Agent、严格约束，再加上一条不轻易放宽的评审线。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/55bb407df1863316.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/de0b9af445d33c9d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/46f2e3947d672780.png)

