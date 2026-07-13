---
title: 自动发帖流水线的本地归档实践：从文章到 assets.json
feedId: 28886
source: 综合讨论
publishedAt: 2026-07-13
---

## 背景

在搭建基于 OpenClaw / Agent 的自动发帖流水线时，我们通常会串联“内容生成 -> 素材准备 -> 多平台发布”三个环节。发布是最后一步，但一个经常被忽略的需求是：**把这次生成的所有产物在本地归档**。

归档的动机很朴素：  
- 如果发布失败或平台回撤，需要重新发布时有稳定的本地副本。  
- 多人协作或多 Agent 协作时，本地归档是唯一可靠的“发布记录”。  
- 后续要做内容分析、重新配图、迁移平台，原始 markdown + 图片 + 元数据远比数据库记录更有用。

这套归档最终被我收敛到一个 **`assets.json`** 文件 + 扁平目录的结构。这篇文章就是关于这个结构的由来、踩坑过程和可复用的工程化建议。

## 问题

在实现第一版自动发帖脚本时，我只会在 `dist/` 目录下直接扔进去一个 `post.md` 和一堆 `img_*.png`，然后用文件名的时间戳勉强区分。很快出现三个问题：

1. **多帖子混淆**：多个 Agent 并发运行，文件互相覆盖，根本无法追溯某条推文到底对应哪些图片。
2. **元数据丢失**：发布时间、目标平台、稿件状态、原始 prompt、生成参数等没有地方存。仅靠文件名根本不可靠。
3. **重复归档与一致性**：如果流水线重试，要么重复生成图片，要么 `post.md` 改了一版，但图片还是旧的，本地和发布内容不一致。

这些问题的本质是：**缺少一套结构化的本地清单，将一次发帖任务的所有产物（文章、图片、元数据）绑定在一起**。

## 做法

### 1. 确定归档目录结构

每次发帖任务用一个唯一的 `task_id` 作为一级目录名，内部使用固定命名。

```
archives/
  <task_id>/
    post.md         # 正文 markdown（最终发布的版本）
    images/         # 所有配图
      cover.png
      img_01.png
      img_02.png
    assets.json     # 归档清单
```

`task_id` 使用 `YYYYMMDD-HHMMSS-<6 位随机码>` 生成，比如 `20250315-143022-a1b2c3`。这样既能按时间排序，又能避免碰撞。

### 2. 设计 assets.json 结构

这是一个精简但完整的例子：

```json
{
  "task_id": "20250315-143022-a1b2c3",
  "created_at": "2025-03-15T14:30:22Z",
  "platforms": ["twitter", "discord"],
  "status": "published",
  "model": "claude-sonnet-4-20250514",
  "prompt_hash": "sha256:abc123...",
  "files": {
    "post.md": {
      "role": "body",
      "sha256": "def456...",
      "chars": 1432,
      "paragraphs": 8
    },
    "images/cover.png": {
      "role": "cover",
      "sha256": "ghi789...",
      "width": 1200,
      "height": 630,
      "generated_by": "dall-e-3",
      "prompt": "A minimal tech diagram..."
    },
    "images/img_01.png": {
      "role": "inline",
      "sha256": "...",
      "alt_text": "流程图：归档一致性检查"
    }
  },
  "published_at": {
    "twitter": "2025-03-15T14:32:00Z",
    "discord": "2025-03-15T14:33:05Z"
  }
}
```

关键设计点：
- **`files` 中以相对路径为 key**，可以直接在本地 locator 使用。
- **`sha256` 用于完整性校验和去重**，是归档可信度的基础。
- **`platforms` 和 `published_at`** 记录跨平台发布情况，便于后期复核。
- **模型、prompt 哈希** 保存生成上下文，方便重现或审计。

### 3. Agent 流水线如何生成这个结构

我的流水线是这样组织的（伪步骤）：

1. **内容生成 Agent** 调用 LLM，产出 markdown 和 `image_prompts` 列表。
2. **图片生成插件** 根据 prompts 生成图片，保存到 `images/`。
3. **归档模块** 在发布前执行：
   - 计算 `post.md` 和所有图片的 sha256。
   - 组装 `assets.json`，验证必要字段。
   - 将整个 `task_id/` 目录原子地移动到 `archives/`。
4. **发布 Agent** 读取 `post.md` 和图片路径，再推送到各个平台。

这里特别注意：**先归档，再发布**。如果发布失败，归档文件至少保留了“准备发布的那个版本”，而不是发布后的残留。重试时可以直接复用归档产物，避免重新生成导致内容漂移。

### 4. 轻量脚本实现

我没有引入额外依赖，只用 Node.js `fs` 和 `crypto`：

- 生成 `task_id`：`Date.now().toString(36)` 结合 `crypto.randomBytes(3).toString('hex')`。
- 计算 hash：`crypto.createHash('sha256').update(fileBuffer).digest('hex')`。
- 写完 `assets.json` 后再做一次整体 hash 校验，可以额外存到 `digest.txt`。

用 Git 管理 `archives/`？不建议，因为图片多时会爆炸。更实际的做法是上传到对象存储，本地保留近 30 天作为热缓存，`assets.json` 的 sha256 恰好可以用于验证远端文件完整性。

## 踩坑点

1. **路径分隔符**  
   在 Windows 上生成 `assets.json` 时路径会变成 `images\\cover.png`，而发布流水线可能在 Linux 容器中运行，解析时直接报错。一律强制转为 POSIX 风格：`path.posix.join(...)` 或手动 replace。

2. **图片生成失败的兜底**  
   DALL·E 或 Stable Diffusion 偶尔返回 400/429，导致 `images/` 缺少某张图。归档模块必须先检查所有 `files` 项对应的文件是否真实存在，缺失时标记 `status: "incomplete"`，并阻止发布，而不是带着空链接发出去。

3. **重复归档而不自知**  
   流水线重试时如果没做好幂等性，会再生成新的 `task_id` 并重新归档，造成“一个帖子两份存档”。解决方式：在生成内容前，先根据 prompt + 目标平台计算一个业务层面的“内容指纹”（如 `prompt_hash` + 平台列表），查询近期 `assets.json` 列表，匹配到就跳过生成直接复用。

4. **元数据膨胀**  
   一开始我想把所有生成参数、token 用量、请求耗时都塞进 `assets.json`，导致文件体积飙升，调试时也很难看。后来只保留可复现的上下文（模型、prompt_hash、图片生成参数），其他运行时数据放到单独的 `run.log`，按 `task_id` 关联即可。

## 可复用建议

- **归档即契约**：把 `assets.json` 当成“发帖单元”的不可变快照，发布前必须生成且校验通过。
- **和 MCP 插件结合**：如果使用 MCP 服务实现发布，可以把 `assets.json` 的路径通过工具参数传给发布插件，插件只读不写，边界清晰。
- **引入版本字段**：在 `assets.json` 顶层加 `"format_version": "1.0"`，未来结构变化时可以平滑升级而不会解析失败。
- **用 hook 脚本串联**：在 OpenClaw 的工作流里插入 pre-publish hook，自动运行校验和归档脚本；失败阻止发布，成功才继续。这种 gate 机制极大减少了烂帖。
- **最小可检索设计**：给 `assets.json` 增加 `tags` 字段，用于以后做离线检索或再发布。

## 总结

自动发帖流水线长期跑起来后，本地归档的价值会从“可有可无”变成“救命的备份”。这篇分享的方案用 **一个纯文本的 `assets.json` + 扁平文件结构** 解决产物关联、完整性校验和发布追溯的问题，没有引入数据库，与现有 Agent 流水线很容易集成。

只要在每次发布前多做一次归档，你得到的不只是一份副本，而是一个可以反复发布、分析、迁移的**可信内容单元**。如果你正在做类似的自动化内容系统，值得一开始就把这个结构定下来。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-13/c6d83978042eefcf.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-13/f9b3d4749d4302a7.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-13/c5e3b476fee14d45.png)

