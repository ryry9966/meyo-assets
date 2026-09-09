---
title: OpenClaw Skills 机制：如何让 AI 助手按需加载能力
feedId: 36740
source: 综合讨论
publishedAt: 2026-09-09
---

## 背景

OpenClaw 的 Agent 跑在一个有限的上下文窗口里。随着接入的能力越来越多——浏览器控制、定时任务、消息通道、各类 MCP 工具——一个朴素做法是把所有工具说明和操作手册都塞进 system prompt。结果就是：常驻说明占满上下文，模型注意力被稀释，token 开销上涨，真正干活的指令反而被挤到边缘。

Skills 就是为这个问题设计的：能力按需加载，而不是常驻。

## 机制核心：渐进披露

一个 Skill 本质是「一个文件夹 + 一个 SKILL.md」，加载分三层：

1. **常驻层**：只有每个 Skill 的名称和一句 description 进入 system prompt，成本极低。
2. **触发层**：模型判断当前任务命中某条 description 时，才去读该 Skill 的完整 SKILL.md。
3. **执行层**：SKILL.md 里可以引用更多参考文件（脚本、模板、长文档），执行时再按需读取。

也就是说，一百个 Skill 的常驻成本可能只相当于几段文字，真正展开的只有当前用到的那一两个。

## 做法：写一个能被正确触发的 Skill

1. 在 workspace 的 `skills/` 目录下建 `your-skill/SKILL.md`；
2. 写 frontmatter：`name`、`description`，按需加 metadata 做环境门控（`requires.bins`、`requires.env`、`os`）；
3. 正文只写「这个能力怎么用」的操作性内容，不要写背景故事；
4. description 是触发 key——用「什么时候需要我」的句式，而不是功能罗列。

示意结构：

```markdown
---
name: pdf-extract
description: 当用户要求从 PDF 提取表格或文本时使用。依赖 pdfplumber。
metadata:
  requires:
    bins: ["python3"]
---
## 步骤
1. 先确认文件路径存在
2. 用 pdfplumber 按页提取
...
```

5. 重启会话后用自然语言验证：问一个明确该命中的问题，确认模型确实去读了 SKILL.md，而不是靠猜。

## 踩坑点

- **description 写成功能清单**：「支持 PDF、Word、Excel」——模型无法判断何时该用它。改成任务导向：「当需要从 PDF 提取内容时」。
- **SKILL.md 过长**：细节全塞正文，等于变相回到全量加载。长内容拆成子文件，正文只留入口和步骤。
- **没做环境门控**：依赖某个二进制或环境变量但不声明，触发后第一次执行才报错，白白浪费一轮交互。用 `requires` 提前过滤。
- **语义重叠**：两个 Skill 的 description 高度相似，模型会随机命中其中一个。要么合并，要么在描述里划清边界。
- **写完不测**：触发条件全靠猜。每次改动后跑一遍最小触发用例，是唯一可靠的质量手段。

## 可复用建议

- 把 Skill 当 API 设计：description 是签名，正文是文档，环境依赖是显式声明。
- 一句话原则：**常驻信息最小化，触发条件明确化，执行细节外置化**。
- 团队场景分层：通用 Skill 放共享目录，个人习惯类放 workspace，避免互相覆盖。
- 定期做「触发审计」：翻近期会话日志，找出从未触发的 Skill 和该触发没触发的场景，据此收敛 description 措辞。

## 总结

Skills 的价值不在「多了个插件系统」，而在于把上下文当成稀缺资源来管理：用一句 description 换取一次按需加载的机会。写好一个 Skill 的关键不是内容多全，而是让模型在最对的时刻，只读最需要的那份说明。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/35c5ff5797a07131.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/79bcf47382e1cbc1.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/6bd85e94d3b8ee7c.png)

