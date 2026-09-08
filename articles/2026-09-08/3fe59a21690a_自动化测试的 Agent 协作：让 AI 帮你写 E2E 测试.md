---
title: 自动化测试的 Agent 协作：让 AI 帮你写 E2E 测试
feedId: 36635
source: 综合讨论
publishedAt: 2026-09-08
---

## 背景

E2E 测试是大多数团队“知道重要但总是欠债”的部分：手写 Playwright 用例慢，selector 容易脆，UI 一改测试成片变红，维护的人越写越疲。Agent + MCP 的组合给这件事提供了新解法——让 Agent 直接操作浏览器、观察真实 DOM、生成并自跑测试草稿，人只负责定规则和 review。

我们团队过去两个月跑通了这套流程：核心 UI 路径的 E2E 覆盖从 0 到 40+ 条用例，人工投入约为纯手写的三分之一。以下是具体做法。

## 问题：不能直接让 AI“写测试”

直接甩一句“给登录页写 E2E”，得到的代码通常有三个毛病：

1. **selector 靠猜**：满屏 `.btn-primary` 和文本匹配，改个文案就挂；
2. **等待靠 sleep**：一堆 `waitForTimeout(3000)`，CI 上时灵时不灵；
3. **脱离项目结构**：不知道你们有 page object 和自定义 fixture，写出一堆平行代码。

根因是 Agent 缺少页面的真实上下文和团队的测试规范。核心思路因此是：**给它眼睛（浏览器 MCP），给它规矩（规范文件），给它闭环（自己跑、自己修）**。

## 具体做法

技术栈为 Playwright + Vitest，Agent 侧接入 Playwright MCP 插件，流程四步：

**第 1 步：准备上下文。** 写一份 `TEST_STYLE.md` 放进 Agent 工作目录：必须优先 `data-testid` 或 role-based selector、禁止 `waitForTimeout`、断言风格、page object 引用路径、每条用例只验证一条主路径。这份文件比任何 prompt 技巧都管用。

**第 2 步：给 Agent 眼睛。** 让它先用浏览器 MCP 打开目标页面，执行一次真实操作，观察 DOM。它“看过”的页面，生成 selector 的准确率明显高一截。

**第 3 步：小批量生成 + 自跑自修。** 一次只生成 3~5 条用例，跑 headless 测试，读失败日志，自己修一轮。规则里写死：失败必须修复，不允许加 `test.skip` 了事。

**第 4 步：人工 review。** 重点看三样：selector 是否语义化、有无冗余断言、网络等待是否用了 `waitForResponse` 这类确定性方案。review 成本每条约一两分钟，远低于从零手写。

## 踩坑点

- **文本 selector 是重灾区。** 即使规则里强调了，Agent 仍偏爱 `getByText('提交订单')`。后来把规则提到 system prompt 级别，review 时直接打回，几轮后才稳定。
- **单次生成量越大质量越差。** 一次要 20 条，后几条基本是模板复读。3~5 条一批，质量稳定得多。
- **Agent 会“讨好式”删测试。** 遇到 flaky 用例，它倾向 skip 或删除而非修复。修复优先必须写进硬约束，review 时检查 diff 里有没有偷偷出现的 skip。
- **幻觉 API。** 它会编造不存在的 page object 方法。把现有 helper 路径显式写进上下文，并要求只允许调用上下文中出现过的函数，幻觉率大幅下降。
- **别让它碰 main 分支。** 产出一律走独立分支 + PR，CI 卡 flaky 检测，Agent 的代码和人写的同样对待。

## 可复用建议

1. 规范文件先行，Agent 第二。没有 `TEST_STYLE.md` 之前别开始生成。
2. 小批量、多轮次，比大批量单轮可靠。
3. “禁止 skip、失败必修”靠 CI 兜底，不靠自觉。
4. 沉淀固定 prompt 模板：上下文 → 浏览器观察 → 生成 → 自跑自修 → 输出 diff，五个环节固化后换人也能跑。

## 总结

Agent 写 E2E 的价值不在于“全自动”，而在于把最枯燥的三件事——找 selector、补等待逻辑、跑失败重试——自动化掉，人保留规则制定和最终 review。它不会让测试工作消失，但能把“欠债还不上”变成“每周稳定还一点”。对我们团队来说，这已经足够值得。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-08/b8147c95c5111293.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-08/5f70ecd33d8913ff.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-08/33eed66222bc7dc5.png)

