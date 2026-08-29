---
title: OpenClaw Skills 机制：让 AI 助手按需加载能力，而不是一次性塞满上下文
feedId: 35311
source: 综合讨论
publishedAt: 2026-08-30
---

## 背景

OpenClaw 的 Skills 不是简单的“更多工具”，而是一种把能力封装成可索引目录的机制。一个 Skill 通常是一个目录，里面包含 `SKILL.md` 元数据、正文，以及可选的 `scripts/`、`references/` 文件。运行时先只读取元数据，只有在判断相关时才加载正文和依赖，类似“能力目录 + 按需展开”。

以下路径以本地安装为例，不同版本可能略有差异，建议先确认你自己的 `~/.openclaw/skills` 或项目内 `.openclaw/skills` 位置。

## 问题

如果把所有工具说明、脚本流程、checklist 都堆在 system prompt 或常驻工具里，会很快遇到几类问题：

- 上下文占用持续增加，延迟变高；
- 工具描述互相干扰，模型容易选错；
- 无关能力影响当前任务判断；
- 维护成本高，改一个小流程都要动全局 prompt。

MCP 适合暴露外部服务，但内部流程、脚本化操作、固定排查步骤，并不适合全部做成常驻工具或长 prompt。Skills 就是为这部分能力做一个延迟加载层。

## 做法/步骤

**1. 建立技能目录**

```
~/.openclaw/skills/csv-report/
  SKILL.md
  scripts/
  references/
```

团队共用的可以放在项目 `.openclaw/skills/` 下，纳入版本管理。

**2. 写清楚元数据**

`SKILL.md` 顶部用 YAML front matter 描述触发条件：

```yaml
---
name: csv-report
description: >
  Use when user asks to summarize, clean, or convert CSV files.
  Triggers: .csv, comma-separated, table file.
  Do not trigger on .xlsx or binary files.
---
```

`description` 是索引核心。它决定什么时候加载技能，所以要把触发词、任务类型和排除条件都写清楚。

**3. 正文只放工作流**

正文不要贴大段代码或长文档，保持在 80–150 行左右。例如：

```markdown
# CSV Report

## When to use
- Input is a CSV or comma-separated table file.

## Steps
1. Check file encoding with `file -I <input>`.
2. Run:
   `python3 {SKILL_DIR}/scripts/clean_csv.py --input <input> --output <output>`
3. If encoding errors appear, read `references/encoding-fixes.md`.

## Output
Return JSON with row_count, bad_rows, output_path.
```

**4. 脚本外置，保持可测试**

把确定性操作放到 `scripts/`，并给它执行权限：

```bash
chmod +x scripts/clean_csv.py
```

脚本最好通过参数或 stdin/stdout 交互，不要依赖模型反复补全参数。退出码非零应表示失败。

**5. 引用文件按需加载**

细节内容放 `references/`，正文只写“什么情况下读哪个文件”。这样正常任务不会白白加载长文档。

**6. 测试触发**

至少做两轮测试：

- 强提示：`Use csv-report skill to process this CSV file.` 确认技能被显式加载；
- 弱提示：`帮我看下这个 csv。` 看是否自然触发。

两轮结果都符合预期，再进入实际使用。

## 踩坑点

- **description 过宽**：只写 `data processing` 会频繁误触。要写清文件类型、任务类型和排除条件。
- **YAML 缩进错误**：技能可能静默注册失败。不要用 tab，写完可以用 `yaml.safe_load` 校验 front matter。
- **相对路径错误**：脚本里用相对当前工作目录的路径很容易跑偏。优先使用 `{SKILL_DIR}` 或 `Path(__file__).parent`。
- **正文过长**：如果正文超过 150 行，按需加载的意义会明显下降。细节外置到 `references/`。
- **编辑后不生效**：本地技能可能有索引缓存。修改后重载技能列表，或新建会话验证。
- **与 MCP 工具冲突**：如果技能内部也调用 MCP，要写明优先级，例如“优先使用本 Skill 脚本；若输出不符合预期，再转 MCP 工具 Y”。

## 可复用建议

1. 一个 Skill 只解决一个能力边界。`csv-report` 不要顺便做 PDF 合并。
2. 用“触发词 + 排除词”维护 `description`。每遇到一次误触，就补一条 `Do not trigger on ...`。
3. 脚本保持纯输入/输出，参数化，避免交互式提问。失败就给非零退出码和明确错误信息。
4. 技能目录纳入版本管理，团队协作时先评审 `description`。索引错误比正文错误更难排查。
5. 如果 OpenClaw 支持执行允许名单，只放行技能目录，禁止任意 shell。

## 总结

OpenClaw Skills 的价值不在“增加更多能力”，而在“延迟加载”和“可维护封装”。元数据是触发入口，正文是执行流程，脚本是确定性操作。保持描述克制、正文短小、脚本可测试，才能让 AI 助手按需加载能力，而不是每次启动都全量携带。下一步可以把一个高频手工流程改成一个 Skill，再让 MCP 只负责外部服务，内部流程归 Skills。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/ba592c2929e677ec.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/0ab7d826e90f4812.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/bb0a9a11997f709e.png)

