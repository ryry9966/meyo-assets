---
title: 为自动化发布流水线建立本地归档：文章、图片与 assets.json
feedId: 28961
source: 综合讨论
publishedAt: 2026-07-14
---

## 背景

在 OpenClaw 与 MCP 生态里，越来越多同学把内容生成、多平台分发做成自动化流水线：Agent 产出文章，调用 mcp-server-wechat、mcp-server-zhihu 等工具一键发布。但发布流程常常只关注“发送成功”，忽略了本地留档。没有归档意味着：

- 无法回溯某次发布的原始内容（尤其是图片已被 CDN 清理或替换后）；
- 想修改重发历史文章时，找不到当时的确切版本；
- 排查多平台展示差异时，缺少一致的本地基准。

为此，我们在发布 Pipeline 中增加一个**本地归档环节**，在内容推送前把所有素材稳定保存到本地目录，并用 `assets.json` 记录结构化元数据。

## 问题拆解

一次典型发布包含：Markdown 正文、多张网络图片（AI 生成图或外链）、各平台适配参数（标题/摘要/标签）。如果仅靠日志或脚本临时变量，不足以构成可复现的快照。

归档要解决三个问题：

1. **Markdown 与图片的原子保存**——图片必须从远程下载到本地，且 Markdown 中的引用改为相对路径，保证离线可读；
2. **结构化的元数据记录**——需要一个 `assets.json` 存储标题、slug、生成时间、目标平台、关联资源清单等；
3. **集成进发布 Pipeline**——归档应在内容生成后、发布前执行，且归档失败应终止发布（fail-fast），避免“发出去了但本地没存档”。

## 实现步骤

### 1. 确定归档目录结构

以 `archive/` 为根，按日期加 slug 组织：

```
archive/
  2025-02-01-understanding-mcp/
    article.md
    images/
      cover.png
      diagram.png
    assets.json
```

slug 从标题生成，注意处理特殊字符：用正则替换非字母数字下划线为连字符，并转为小写。

### 2. 构造归档工具（MCP Tool 或独立脚本）

如果使用 OpenClaw + MCP，可以写一个本地 MCP 工具 `local_archive`，接收参数：

- `markdown_content`：原始 Markdown 文本
- `image_urls`：图片 URL 列表（或从 Markdown 中解析）
- `metadata`：包含 title、slug、platforms、tags 等

工具内部逻辑：

1. 创建目录 `archive/{date}-{slug}/images/`；
2. 遍历 `image_urls`，下载图片到 `images/`，文件名使用 URL 哈希加原始扩展名，避免冲突；
3. 将 Markdown 中的远程图片 URL 替换为相对路径 `./images/{filename}`；
4. 写入 `article.md`；
5. 组装 `assets.json` 并写入，记录生成时间戳、图片映射关系、原始 URL 等。

也可用 Node.js/Python 脚本实现，通过 `child_process` 在发布 Pipeline 中调用。

### 3. 图片下载的健壮性

远程图片下载容易踩坑：

- **防盗链 / UA 限制**：需添加 User-Agent、Referer（有时可留空）；
- **重定向**：开启 HTTP 重定向跟随；
- **超时与重试**：设置连接超时 10s，读取超时 15s，失败重试 2 次；
- **文件类型判断**：优先根据响应的 `Content-Type` 确定扩展名，若缺失则从 URL 后缀猜测，最后回退为 `.png`；
- **重复图片**：用 `hashlib.sha256(url.encode()).hexdigest()[:12]` 生成短标识，配合计数器防止极低概率冲突。

### 4. assets.json 设计

推荐一个可扩展的结构：

```json
{
  "title": "理解 MCP 协议",
  "slug": "understanding-mcp",
  "created_at": "2025-02-01T10:30:00Z",
  "platforms": [
    {"name": "wechat", "post_id": null, "publish_time": null},
    {"name": "zhihu", "post_id": null, "publish_time": null}
  ],
  "images": {
    "cover.png": {
      "original_url": "https://cdn.example.com/cover.png",
      "hash": "a1b2c3d4"
    }
  },
  "tags": ["MCP", "OpenClaw"],
  "version": "1.0"
}
```

`post_id` 在发布成功后通过回调或后续脚本回填，形成闭环。字段命名保持清晰，建议用 JSON Schema 做校验，防止脏数据污染归档库。

### 5. 集成到 Pipeline

发布流程示意：

1. 内容生成（Agent）→ Markdown + 图片 URL 列表
2. 调用 `local_archive` → 归档成功返回本地路径
3. 若归档失败（写权限、磁盘满、图片下载异常等），**抛出错误终止后续发布**
4. 对每个目标平台，读取归档的 `article.md` 和 `images/`，进行平台适配后发送
5. 发布成功后（可选），更新对应平台的 `post_id` 和 `publish_time`

这样保证了归档和发布数据源的一致，也为调试提供了精确的本地基线。

## 踩坑点

- **图片链接直接替换不处理编码**：如果 Markdown 里的 URL 包含中文或空格等已编码字符，下载时注意保持，本地文件名应使用解码后的可读名称（但要防路径穿越）；
- **assets.json 被手动编辑后破坏结构**：最好在读取时做容错解析，并使用 JSON Schema 校验，发现不规范时给出 warning，但默认不阻塞发布（视严格度调整）；
- **并发发布冲突**：如果同时多个 Pipeline 写同一个 slug 目录（极少见），使用文件锁或基于时间的微秒后缀；
- **slug 重复覆盖**：同一天内可能发布同一标题的不同版本，推荐在 slug 后追加短哈希 `-a1b2`，并在 assets.json 中记录 revision。

## 可复用建议

- **封装成独立工具包**：将归档逻辑做成 CLI 或 MCP Server，发布到内部工具仓库，供任何内容 Agent 调用；
- **结合 Git**：每次归档后自动 `git add` 并 commit，形成内容仓库的版本历史，方便 diff 和回滚；
- **归档校验环节**：在正式发布前，加一个“归档完整性检查”，比如确认 `article.md` 存在、所有图片文件存在且大小 > 0；
- **扩展 frontmatter**：可以在 `article.md` 顶部写入 YAML frontmatter，存放部分 metadata，这样用 Obsidian 等工具也能直接预览，而 `assets.json` 依然保留完整结构化数据。

## 总结

为自动化发布流水线增加本地归档，是一次成本极低但收益持久的工程投入。它既解决了“历史内容不可追溯”的痛点，又为后续的内容审计、多平台一致性对比、甚至重新发布提供了可靠基础。只需几十行脚本和一份清晰的目录规范，就能让你的 Agent 发布系统稳健一个量级。

下次你的流水线因为平台接口变更导致格式异常，或需要找回一年前的某篇文章时，你会感谢当初那个加归档步骤的决定。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-14/2295db859330bfce.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-14/779bc08ddf6acc28.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-14/69bfbf784cccd810.png)

