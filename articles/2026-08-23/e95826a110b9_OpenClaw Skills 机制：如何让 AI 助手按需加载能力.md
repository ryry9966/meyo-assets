---
title: OpenClaw Skills 机制：如何让 AI 助手按需加载能力
feedId: 34315
source: 综合讨论
publishedAt: 2026-08-23
---

## 背景

在 OpenClaw 上扩展 AI 能力，常见两条路：把资料塞进 system prompt，或挂 MCP 工具。前者开始时简单，能力一多问题就来了：20 个 skill 描述、数百行文档常驻上下文，token 成本上升，模型注意力被稀释，指令之间还会互相干扰。

OpenClaw 的 Skills 机制更适合这种场景：每个能力一个目录，入口是 `SKILL.md`，由 frontmatter 提供简短描述。OpenClaw 索引这些描述，不在对话中全量加载；当任务匹配时才把对应 skill 拉进上下文。这样能力扩展从“预加载全部”变成“按需加载”。

不过，如果不做约束，按需加载也会退化成伪按需。工程上要控制两个点：触发精度和加载体积。

## 做法/步骤

以下基于 OpenClaw 的 Skills 目录约定，具体配置项请以当前版本为准。

**1. 目录结构**

我习惯在项目下建立：

```text
.openclaw/skills/
  git-commit-analysis/
    SKILL.md
    scripts/
      analyze_commits.py
    references/
      commit-format.md
```

`SKILL.md` 是唯一入口，脚本和长文档都不进主上下文。

**2. 写 frontmatter**

```yaml
---
name: git-commit-analysis
description: 分析最近 N 条 git 提交，按类型归类和总结。仅在用户要求分析提交历史、生成 changelog 时使用。
when_to_use: user asks about commit history, release notes, or changelog
version: 0.1.0
---
```

描述里同时写“做什么”和“什么时候不用”，能减少误触发。

**3. 入口保持精简**

`SKILL.md` 只写执行步骤：

```markdown
1. 运行 `python3 scripts/analyze_commits.py --limit 20`
2. 若用户需要详细格式说明，再读取 `references/commit-format.md`
3. 输出 Markdown 表格，不要改写提交原意
```

长内容按需读取，不需要一开始就进上下文。

**4. 注册并测试**

在 OpenClaw 配置中把 `skills` 指向该目录，重启或重新索引后，用固定用例测试：“帮我总结最近 20 条提交”。观察模型是否正确加载 skill，而不是直接自己写脚本。

## 踩坑点

- **描述太宽**：写“帮助用户处理 git”会被任何 git 问题触发。描述要写触发条件，最好加排除项，如“不用于创建分支或解决 merge 冲突”。
- **脚本没有执行权限**：`scripts/` 下的脚本需要 shebang 和 `chmod +x`，否则模型调用时报权限错误，然后开始即兴写命令。
- **相对路径错乱**：脚本里不要用 `os.getcwd()` 找 references，应基于 skill 目录定位。OpenClaw 不一定把 CWD 设在 skill 目录。
- **缓存不刷新**：修改 `SKILL.md` 后仍在用旧描述，先重启或清理索引，再测试触发。
- **入口过载**：超过 150 行的 `SKILL.md` 通常说明应该拆到 `references/`。

## 可复用建议

1. **一个 skill 只做一件事**：宁可有 10 个小 skill，不要 1 个万能 skill。
2. **描述用“触发/排除”句式**：一句话说清何时用，一句话说清何时不用。
3. **入口控制行数**：`SKILL.md` 保持在 80–120 行以内，长文档拆到 `references/`。
4. **提供 smoke test**：每个 skill 写一条固定 prompt，例如“分析最近 10 条提交”，用于回归测试。
5. **版本化**：skill 目录纳入 git，frontmatter 中写 `version`，方便排查行为变化。

## 总结

OpenClaw Skills 的价值不在“能加载能力”，而在“能控制加载边界”。把入口写短、触发写准、长内容拆出去，才能避免按需加载名存实亡。对工程化团队来说，它更适合作为可 review、可测试、可版本化的能力单元，而不是又一个存放提示词的位置。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/17f423f75c829ffc.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/7c6dc2868bb66e32.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/7d04f0f848be390d.png)

