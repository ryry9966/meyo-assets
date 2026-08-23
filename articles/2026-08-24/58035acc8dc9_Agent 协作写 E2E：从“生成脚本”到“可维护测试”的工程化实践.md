---
title: Agent 协作写 E2E：从“生成脚本”到“可维护测试”的工程化实践
feedId: 34434
source: 综合讨论
publishedAt: 2026-08-24
---

## 背景

E2E 测试能兜住核心链路，但维护成本一直很高：前端一改版，选择器失效；断言写得太宽，测了个寂寞；录制回放工具生成的脚本又长又脆。引入 LLM 后，不少人尝试让一个 Agent 从零写 Playwright 测试，结果往往是 demo 能跑，进项目就崩。

问题不在“AI 不会写测试”，而在单个 Agent 承担了太多角色：既要理解页面结构，又要设计用例，还要处理运行失败。没有约束和协作，生成出来的脚本自然不可维护。

## 问题：单个 Agent 直接生成的典型失败

1. **上下文不足导致编造选择器**：只给页面截图或部分 DOM，Agent 会猜 class、猜 nth-child，跑起来就崩。
2. **目标漂移**：倾向于生成 happy path，忽略等待、隔离、权限状态和失败分支。
3. **可维护性差**：长函数、魔法字符串、没有 Page Object，测试代码比手写还难改。

所以关键不是让 AI 一把梭，而是把流程拆开，让不同 Agent 在受控环境下协作。

## 做法：三个 Agent + Playwright MCP

在 OpenClaw 里，我通常用三个 Agent 完成一轮 E2E 测试生成，底层通过 Playwright MCP 暴露浏览器能力。

**环境准备**  
启动本地 dev server，给 Playwright MCP 配置 headless、trace 和截图目录。不要让 Agent 直接访问生产环境，避免脏数据。

**Agent A：探索**  
给定目标页面和操作路径，只允许读取 DOM、快照、触发导航，不允许写测试。输出结构化页面对象草稿：关键交互元素、状态变化、输入输出、候选 data-testid。如果页面比较复杂，让它分区域扫描，避免一次性拉全量 DOM。

**Agent B：生成**  
基于探索记录和仓库内现有测试规范生成代码。约束要明确：只使用 `page.locator`，优先 `data-testid`，禁止 CSS 层级选择器；每个用例至少一个业务可见断言；等待交给 web-first assertion，不手写 `sleep`；状态清理放 `beforeEach`。

**Agent C：审查修复**  
运行生成的测试，读取失败信息、截图和 trace。最多修复 3 轮，修复必须改测试代码或报告前端缺陷，不允许静默跳过。跑完后输出 diff 和“需要前端补充 testid 的清单”。

**人工确认**  
只审查 diff，不审原始脚本。重点关注业务断言是否真实、测试是否真正走到目标分支。

## 踩坑点

- **MCP 工具超时/快照过大**：页面复杂时 DOM 快照会撑爆上下文。让探索 Agent 分区域扫描，限制输出长度。
- **选择器不稳定**：前端 class 哈希化后，Agent 爱用 `nth-child`。提前约定 `data-testid`，或者让 Agent 输出“需要补充 testid”的清单。
- **登录态难处理**：每次 UI 登录会拖慢测试且不稳定。让 Agent 在 `beforeEach` 通过 API 种 token，而不是走登录页。
- **断言过宽**：比如只断言“页面出现某文字”，可能测试没走到关键分支也通过。要求断言与用户可见的状态变化相关。
- **不要盲目信任通过**：通过不代表测到了。保留 trace 和截图，首轮人工抽查。

## 可复用建议

1. 把“生成测试”当作“生成可审查资产”，不追求全自动。
2. 先选 3-5 条核心链路切入，别全量铺开。
3. 把 `data-testid` 约定纳入前端开发规范，用 code review 守住。
4. CI 里把 Agent 生成的测试跑在独立 job，先不阻塞合并，观察稳定性。
5. 每次改版后跑一遍“探索-生成-审查”流程，让 Agent 更新选择器，再人工确认 diff。

## 总结

Agent 协作写 E2E 的价值，不是省掉测试工程师，而是把“探索页面、起草脚本、失败修一轮”这种高耗时低决策的活分给 Agent。人保留对业务断言和可维护性的判断。落地关键在约束：工具约束、数据属性约束、审查轮次约束、CI 策略约束。约束越明确，Agent 生成的测试越接近工程可用。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/4e4c633d8aa4be92.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/ee31eff393d0b586.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/bbb57671236c4282.png)

