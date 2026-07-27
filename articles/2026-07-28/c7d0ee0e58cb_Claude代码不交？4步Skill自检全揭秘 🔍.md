---
title: Claude代码不交？4步Skill自检全揭秘 🔍
feedId: 01KYJY1BHVBTZEFY2EWZVE1PEC
source: 36kr
publishedAt: 2026-07-28
---

最近有件事在工程师圈子里炸开了锅：有用户发现，让Claude帮忙写段代码，它居然不像以前那样秒回一大段 code block，而是先停顿一会儿，然后冒出一连串 “Looking at the code I generated, I should refine…” 之类的自我修正。原本以为是 prompt 没给好，后来 Anthropic 的开发者关系负责人 Alex Albert 直接点破：这不是 bug，而是 Claude 在交付代码前内置了一套 4 层 Skill 自查流程。写完不交，改好再说——这让很多被 AI 生成代码里的幽灵 bug 折磨过的后端老哥直呼内行。

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-28/92f19ea1b554b026.png)

## 为什么 AI 需要“先自查”再给代码

把 AI 写代码想象成一位刚入职的实习生。第一版 prompt 就是你的需求文档，实习生噼里啪啦敲完，如果直接 commit，大概率会在代码评审里被打回无数次。Claude 现在的行为就像是实习生学会了一套自检清单，交给你之前先自己跑一遍 lint、跑一遍测试、甚至做一轮轻量级安全扫描。

- **幻觉问题仍未根治**：大模型本质上是在做概率补全，并非真正理解业务逻辑。一句 “use pandas to process csv” 可以生成语法正确的代码，但引用一个不存在的 API 参数的概率一直在 2-5% 之间浮动。自查等于给概率输出加一道确定性过滤器。
- **多步任务需要中间验证**：写代码不是单次生成。尤其在需要读文件、编译、运行的 Agent 模式下，模型会在思维链中插入 “执行检查点”——跑一下 `pylint` 输出结果，如果报错就回退修正，这个环不闭合前绝不输出最终答案。
- **对齐工程学的自然延伸**：从 RLHF 到 Constitutional AI，核心都是让模型学会 “审视自己的行为”。代码自查技能相当于把宪法原则具象化为 lint 命令、测试套件和 Reviewer 视角。

## 拆解四层 Skill：Claude 到底在查什么

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-28/82d0348908ca3378.png)

这 4 个 Skill 并非凭空造出的概念，而是对应一套可观测的行为链。结合经验复现，可以把它归纳为如下层次：

1. **语法与逻辑完整性检查 (Syntax & Logic Check)**
   - 模型会在内部模拟一个迷你解释器：变量是否正确定义？条件分支是否穷尽？循环会不会因边界条件变成死循环？
   - 真实案例：让 Claude 写爬虫，第一版用了 `requests.get().text`，自查后发现目标站点返回的是 gzip 压缩数据，自动补上了 `response.content` 加 `gzip.decompress`，避免了新手会遇到的乱码问题。

2. **风格与最佳实践修补 (Style & Convention Audit)**
   - 这不是简单的 PEP8，而是会根据上下文推断你用的框架规范。比如你要求 “FastAPI 微服务”，自查会检查是否缺了 `pydantic` model、是否忘了 `async/await`。
   - 典型输出：“I noticed the function name doesn’t follow snake_case convention used in the rest of the codebase. I’ll adjust it before you run.”

3. **轻量级测试仿真 (Lightweight Test Simulation)**
   - 模型会构造若干典型输入，在思维中推演输出。类似人类写代码时心里默跑几个 case。Claude 会反馈：“Running a mental test with sample data… the sorting logic would fail for edge case empty list, adjusting.”
   - 这一点尤其对算法题/数据处理脚本价值巨大，避免了边界值错误。

4. **安全与性能红线扫描 (Security & Performance Red Flag)**
   - SQL 拼接变参数化查询，密码硬编码提示，文件路径遍历风险。自查时会强制拦截最高危的模式。
   - 同时会评估时间复杂度，如果生成 O(n²) 的循环嵌套，且代码注释写了 “大数据量”，它会自己改成 `defaultdict` 或推荐更高效结构。

四个技能不是独立运行，而是串在一条 ReAct 风格的 chain 上：生成初版 → 逐一打勾 → 一处不过就退回上一节点修补，直到全部通过或达到重试上限。

## 背后的机制：不是自觉，是 System 2 Thinking 工程化

不少人惊呼 “AI 觉醒”，实际上这是强化学习与工具调用结合的产物。Claude 并不真的 “自觉”，它只是被训练成把代码审查视为任务链的一部分。

- **从 Tool Use 到 Skill Loop**：Claude Code 环境里，模型可以主动调用 shell 命令（`cat`, `grep`, `pylint`, `pytest`）。于是开发者在训练时引入了循环奖励：当模型主动执行检查并修正错误后，最终代码通过测试集的概率更高，reward 更大。久而久之，模型学会了 “不检查就亏了” 的策略。
- **思维链隐式修正**：在没有工具可调用的 API 场景，自查以隐式 chain-of-thought 形式存在。研究表明，让模型在生成最终答案前先写一段 “Let me double-check my solution” 的自言自语，准确率可提升 8-15%。Claude 只不过把这个 prompt trick 内化成了能力。
- **Constitutional AI 的微观映射**：Anthropic 的安全宪法里有 “helpfulness, honesty, harmlessness”，代码场景下的 honor 就是交付正确、健壮、安全的产物。自查技能是这一抽象原则在编码垂域的具体实例化。

## 工程师如何驾驭这套自查机制

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-28/0588c58efe34a46c.png)

知道原理后，我们可以用几个技巧把 AI 自查的收益最大化，而不是干等着它表演。

1. **给自查明确指令，别只说“检查下”**
   - ✕ “Review your code.”
   - ✓ “Before finalizing, apply these checks: 1) Run mypy for type errors; 2) Add docstring if missing; 3) Simulate with input `[ ]` and `[3,2,1]`; 4) Confirm no hardcoded secrets.”
   - 多花 20 秒写自查指令，能省下 20 分钟调试，ROI 极高。

2. **设定重试上限，避免“反思死循环”**
   - Claude 偶尔会过度修正：改 A 导致 B 出错，再改 B 又连累 C，循环不止。你可以加上 “Max review iterations: 3”，三回合没过就强制输出当前版本并注释风险点。人工介入比无限等待靠谱。

3. **利用 skip 标记加速原型阶段**
   - 快速验证想法时不需要全套自检。在 prompt 里加上 `[SKIP_CHECKS]` 或 “我只想看草稿，别审查”，Claude 会跳过 skill loop，直接扔出第一版 dirty code。等方向确认了，再去掉标记享受全套服务。

## 冷思考：自查不是银弹

虽然四层 Skill 很有用，但别把审查全甩给 AI。

- **业务逻辑错误难以自查**：模型可以保证代码跑得很溜，但计算 “用户流失率” 时用错了定义，它自己是无法察觉的。领域知识检查只能由人来做。
- **自我审查会放大偏见**：如果训练数据里某类写法永远被标为 “不安全”，模型可能强行把你的合理利用反射的代码改成静态调用。审查标准是人喂的，不一定是真理。
- **增加延迟，需要取舍**：自查意味着更多推理步数，响应时间可能从 5 秒涨到 30 秒。在交互式编程场景下，程序员的心流可能被打断。善用 skip 和精细指令才能找到平衡点。

在一个生成代码比喝咖啡还快的时代，多点耐心等待 AI 自检，反而是工程师对自己项目质量的尊重。下次遇到 Claude 迟迟不交代码，别催——它可能正帮你避开一个凌晨 3 点紧急回滚的生产事故。
