---
title: 低成本图片托管实践：用 GitHub + jsDelivr 搭建可自动化集成的免费图床
feedId: 31344
source: 综合讨论
publishedAt: 2026-08-02
---

## 背景

构建自动化代理、MCP 工具或文档站点时，总会碰到需要托管图片的场景：Agent 生成的图表、Bot 输出的截图、插件的静态资源图标。商业图床要么收费，要么有大量限制，而直接用 GitHub 仓库托管图片加载又慢得令人抓狂。jsDelivr 提供了一个不错的折中：利用 GitHub 作为存储源，通过 CDN 加速分发，并且完全免费。对于小规模、非恶意的自动化项目来说，这套组合足够轻量且可编程性强，很适合接入 OpenClaw 这类追求工程化的代理系统。

这篇文章不鼓吹“永久免费无限流量”，只记录我在实际项目中选型、踩坑、封装复用的一手经验，并给出可落地的自动化集成建议。

## 问题拆解

只用 GitHub 仓库托管图片有两个致命伤：
1. **访问速度**：raw.githubusercontent.com 某些地区加载极慢，间歇性抽风。
2. **无缓存优化**：哪怕浏览器侧有协商缓存，首次请求延迟依然很高，无法支撑高频访问的 Bot 接口。

jsDelivr 作为一个全球 CDN，可以自动拉取 GitHub 上的公开资源并分发。它的地址格式为：
```
https://cdn.jsdelivr.net/gh/{user}/{repo}@{version}/{path}
```
支持语义版本、分支名或 commit hash，甚至连 `@latest` 都可以。这让我们能按需使用固定的发布版本，也方便回滚。

但直接手动管理图片再拼接链接，在自动化场景中太低效。我需要的是一套流程：**Agent 生成图片 → 自动推送至对应 GitHub 仓库 → 立即获得 jsDelivr 加速链接 → 插入 Markdown 或通知中**。下面具体说做法。

## 实践步骤

### 1. 创建“图片源”仓库
新建一个公开的 GitHub 仓库，例如 `assets-cdn`，专门存放图片文件。目录结构按项目或日期划分，例如：
```
assets-cdn/
  ├─ openclaw/
  │   └─ 2025-01-diagram.png
  └─ mcp-tools/
      └─ icon.svg
```
保持仓库干净，避免放无关代码，因为 jsDelivr 会同步整个仓库（有体积和文件数限制，后面会提）。

### 2. 配置自动化上传
自动化上传的触发方式有两种：直接在 Agent 侧通过 GitHub API 推送，或利用 GitHub Actions 做集中处理。后者更适合多人协作，这里给一个精简的工作流思路：

- **使用 GitHub MCP Server**：在 OpenClaw 的代理中集成 `github-mcp-server`，Agent 拿到图片 Base64 后，调用 `create-or-update-file` 工具，写入 `assets-cdn` 仓库的对应路径。提交时 message 带上 `[skip ci]` 避免触发无意义的 CI。
- **若用脚本**：在本地或 CI 环境中，通过 `git clone --depth=1` 后复制文件，执行 `git add . && git commit -m "auto upload" && git push`。这种方式比较粗暴，适合快速原型。

无论哪种方式，核心是保证上传完成后，图片可在 GitHub 上通过 raw URL 访问。但我们需要 jsDelivr 的加速链接。

### 3. 生成 jsDelivr 链接的规范
图片推送到仓库后，CDN 链接一般约定为：
```
https://cdn.jsdelivr.net/gh/{user}/{repo}@main/openclaw/2025-01-diagram.png
```
这里固定使用 `@main` 分支。除非需要精确版本回溯，否则没必要带上 commit hash，因为 jsDelivr 会缓存分支最新的内容（缓存时间约 12 小时，有 API 可强制更新）。为实现即时获取，我们在上传图片后需要主动请求 jsDelivr 的清除缓存接口：
```
https://purge.jsdelivr.net/gh/{user}/{repo}@main/{path}
```
一个 HTTP GET 请求即可触发 CDN 节点更新。虽然官方不建议在生产中频繁使用，但作为自动化流程的一环，上传时调用一次足够安全。

Agent 侧整合逻辑：上传成功后，根据仓库名、分支和文件路径拼接标准 CDN URL，并发起一次 purge 请求，最终返回可直接使用的链接。

### 4. 集成到 OpenClaw 代理的示例
假设在使用 MCP Client 的 OpenClaw 实例中，我们可以这么包装一个工具函数（伪代码）：
```python
async def upload_to_cdn(image_bytes: bytes, path: str):
    # 1. Base64 编码
    content_b64 = base64.b64encode(image_bytes).decode()
    # 2. 调用 github mcp 创建文件
    await mcp_client.call_tool("create_or_update_file", {
        "owner": "your_user",
        "repo": "assets-cdn",
        "path": path,
        "message": "[skip ci] auto upload",
        "content": content_b64,
        "branch": "main"
    })
    # 3. 拼接 cdn url
    cdn_url = f"https://cdn.jsdelivr.net/gh/your_user/assets-cdn@main/{path}"
    # 4. 触发缓存更新
    purge_url = f"https://purge.jsdelivr.net/gh/your_user/assets-cdn@main/{path}"
    httpx.get(purge_url)
    return cdn_url
```
然后 Register 为一个自定义 MCP 工具，Agent 在需要返回图片时调用这个函数即可。

## 踩坑点与风险

1. **仓库大小限制**：GitHub 建议单仓库 1GB 以下，文件大小 100MB 以内。jsDelivr 对单文件有 50MB 硬限制。图片类资源通常不会触达，但长期累积的大量小文件可能让仓库膨胀。建议定期归档老旧目录，或按月份拆分仓库。
2. **缓存与“幽灵”图片**：即使调用 purge 接口，某些边缘节点仍可能延迟数分钟。如果强调即时性，可以在文件名中嵌入时间戳（如 `chart-1704076800.png`），每次上传新图片而非覆盖原文件，彻底绕过缓存问题。
3. **版权与滥用**：jsDelivr 服务条款明确禁止托管大量非网站静态资源。虽然个人小工具通常不会被封，但不要把它当作视频或者大文件分发网盘。另外，公开仓库意味着图片任何人都能访问，注意敏感信息脱敏。
4. **中国大陆访问**：jsDelivr 域名在国内大部分地区可访问，但偶尔会被 DNS 污染或 TCP 重置。如果服务重度面向国内用户，可以在 nginx 侧做一层反代或额外使用国内 CDN 备选。不要完全依赖。
5. **GitHub Actions 配额**：如果用 Actions 做上传或 purge，注意免费仓库的 2000 分钟/月限制。Agent 直接调用 API 比 CI 更节省额度。

## 可复用建议

- **封装成 MCP 工具**：一个 `upload-image` 工具可以让所有 Agent 复用。输入图片 Base64 和目标路径，返回 jsDelivr URL，大大简化图片引用的工作流。
- **统一命名与版本策略**：采用 `项目/年-月/描述-hash.png` 格式，避免文件名冲突，也方便清理。
- **搭配轻量图片处理**：上传前对图片做一次压缩（如 `sharp`），限制宽高，减少流量和存储占用。
- **监控与兜底**：可以在工具中增加重试逻辑和降级方案，例如 CDN 不可用时直接返回 raw.githubusercontent.com 原链接，让调用方自己处理。

## 总结

GitHub + jsDelivr 图床方案不是最优美、也不是最大规模的选择，但它胜在零成本、可编程性强，非常适合 OpenClaw 这类 Agent 工具的图片分发需求。通过 GitHub API 或 MCP 服务化后，Agent 可以无缝完成“生成—上传—获取 CDN 链接”闭环，而开发者几乎不需要额外维护基础设施。同时务必注意缓存策略和仓库限制，做好压图与定期清理，避免踩入滥用陷阱。在对稳定性要求不极端的自动化实践中，这套方案完全站得住脚。

---

