---
title: 让 Agent 替你写 E2E 测试：一套跑得通的协作流程
feedId: 37037
source: 综合讨论
publishedAt: 2026-09-11
---

## 背景

E2E 测试大概是大多数团队"都知道该写、但一直在欠债"的部分：写一条覆盖完整用户路径的用例成本高、选择器易碎、改版后维护烦。最近我们把 Playwright 的 MCP 工具挂到 OpenClaw 的 Agent 上，让它自己操作浏览器、生成并跑通用例，迭代了几个版本后，把做法和教训整理出来。

## 问题

直接让模型"读代码写测试"基本不可用：选择器靠猜、等待靠 sleep、断言对着想象中的 DOM 写。核心原因是缺少真实环境的反馈。反过来，如果 Agent 能真的打开页面、读到可访问性树、执行点击并看到报错，产出质量会完全不同——所以问题不是"能不能让 AI 写测试"，而是"怎么给它闭环"。

## 做法

1. **圈范围**。只挑 3~5 条核心用户路径（登录、下单、关键配置保存），不追求覆盖率，先把欠债最重的地方填上。
2. **给工具**。Agent 挂 Playwright MCP：navigate、click、fill、读取 accessibility snapshot、截图。关键是用可访问性树而不是整页 DOM，token 消耗小，选择器语义也更稳。
3. **定约定**。仓库里放一份 `test-conventions.md`，要求：优先 data-testid、禁用固定 sleep、登录态统一走 storageState fixture、每条用例独立可重跑。Agent 生成前必须先读。
4. **走循环**。Prompt 上强制执行"探索 → 记录真实选择器 → 写用例 → 本地跑 → 修复"的闭环，跑绿之前不许交给人审。
5. **进 CI**。Agent 的产出和人写的代码一样过 CI；失败时把 trace 和截图回喂给 Agent，让它出修复 PR。

## 踩坑点

- **最大的坑是"测试确认 bug"**：Agent 会把当前行为固化成断言，包括错误行为。断言必须人审，重点问一句"这个值本来就该是这个吗"。
- **别一次生成 20 条用例**。实测结果是一堆浅而脆的测试，互相抢数据。每次限定 1~2 条路径。
- **老页面没有 data-testid**，Agent 会退化到文本选择器。要么先补测试 ID，要么明确接受这类用例更易碎并打标。
- **等待问题**：在约定里直接禁掉 `waitForTimeout`，只允许 Playwright 自带的 auto-wait 和 expect 轮询，否则 flaky 率会很难看。
- **上下文成本**：整页 DOM 快照很快撑爆窗口，accessibility tree 加按需截图基本够用。

## 可复用建议

- 工具面越窄越好：5~7 个 MCP 工具足够，工具一多 Agent 反而乱选。
- 把约定文档当作系统的一部分维护，和测试代码一起进 review。
- 用 flaky 率衡量产出：连续两次 CI 绿才算数，不要只看"生成了多少条"。
- 修复回路比生成更值钱：让 Agent 处理 CI 失败的 trace，人只做最终把关，这块的省时最明显。

## 总结

Agent 写 E2E 不是"自动生成测试"的银弹，更像是给团队加了一个不知疲倦、但需要明确边界的初级工程师：给它真实浏览器、一小块范围、清晰约定和闭环反馈，产出是可用的；跳过这些，得到的只是一堆看起来像测试的代码。我们目前的分工是——Agent 负责常规路径的覆盖和日常修复，人负责边界场景和断言正确性。这个比例随着约定文档变厚，还在继续往 Agent 那边倾斜。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-11/142a7cb7b304b5eb.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-11/35ead7e703bbd38c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-11/3d2af4e71332ee0f.png)

