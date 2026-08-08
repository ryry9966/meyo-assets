---
title: GitHub + jsDelivr 图床实战：为你的 Agent 输出免费搭建持久化图片链路
feedId: 32157
source: 综合讨论
publishedAt: 2026-08-08
---

## 背景

在玩 OpenClaw、MCP 服务或者各类自动化 Agent 时，迟早会碰到一个问题——生成的报告、知识卡、运行截图，到底放哪里才能既稳定又免费，还能方便地被 Markdown 文档引用？丢到图床是刚需，但不加选择的用第三方服务，要么很快碰到限流，要么某天链接失效，历史文档一片灰图。

经过几轮折腾，发现 GitHub + jsDelivr 这个组合，只要设计得当，可以充当一个相当稳固的免费图床，尤其适合 Agent 工作流里自动上传、自动引用图片的场景。

## 问题

直接用 GitHub 的 raw 链接有几个硬伤：

1. **速度**：raw.githubusercontent.com 在某些地区访问极慢，甚至间歇性不通。
2. **Content-Type**：GitHub 返回的 MIME 类型有时会是 `application/octet-stream`，导致浏览器不渲染图片而是下载。
3. **缓存策略不可控**：没法主动刷新，更新图片后旧链接可能很久还在用缓存。

jsDelivr 作为免费 CDN，恰好能解决这些问题：全球加速、自动设置正确的 Content-Type、还能通过特定 URL 结构精细控制缓存刷新。

## 做法 / 步骤

### 1. 创建公开图片仓库

新建一个 GitHub 公共仓库，例如 `static-images`。隐私图片别放这里，公开仓库所有人都能访问。

仓库结构建议按项目分目录：

```
static-images/
├── agent-reports/
├── dashboards/
└── knowledge-cards/
```

### 2. 配置上传工具（可选 PicGo，也可全自动化）

如果不打算自己写 Action，PicGo 是最省力的选择。安装后配置 GitHub 图床：

- 仓库：`你的用户名/static-images`
- 分支：`main`
- Token：创建 GitHub personal access token（勾选 repo 权限）
- 自定义域名：`https://cdn.jsdelivr.net/gh/你的用户名/static-images@main`

这样复制出来的链接就是 jsDelivr 格式，省去手动拼接。

### 3. 使用 jsDelivr URL 格式

标准的 jsDelivr URL 是：

```
https://cdn.jsdelivr.net/gh/user/repo@branch/filepath
或者
https://cdn.jsdelivr.net/gh/user/repo@commit-hash/filepath
```

**推荐用 commit hash 而非分支名**。分支名对应的文件更新后，CDN 缓存可能不会立刻刷新，而 commit hash 是一个稳定的不可变版本，完美回避缓存问题。自动上传图片时，可以在上传完成后拿到最新 commit hash，然后拼接链接。

### 4. 在 Agent 工作流中集成

以 OpenClaw 插件为例，如果 Agent 生成了图，可以挂一个上传函数：

```python
def upload_to_cdn(local_path, repo, branch="main"):
    # 读取文件，base64 编码
    # 调用 GitHub Contents API 上传
    # 获取返回的 commit sha
    # 拼接 cdn 链接返回
```

关键点是把返回的 `content.sha` 存入日志，便于后续刷新缓存时使用。

### 5. GitHub Actions 后处理（可选）

可以在仓库里加一个 Action，当新图片 push 后自动压缩、转 webp、生成缩略图，再重新提交。这样不占用本地算力，也让图片体积更小，CDN 加载更快。

## 踩坑点

1. **CDN 缓存刷新**  
   使用分支名链接时，更新图片后可能需要手动刷新缓存。jsDelivr 提供 `https://purge.jsdelivr.net/gh/user/repo@branch/filepath` 接口，提交 GET 请求即可强制刷新。但注意有频率限制，频繁调用会触发 429。更好的办法还是直接用 commit hash。

2. **访问限制与滥用**  
   jsDelivr 没有公开的带宽上限，但如果一天内产生 TB 级流量，有可能会被临时封禁。对个人或小团队来说基本碰不到，但如果你把图大量外链到高流量站点，要留心。另外 GitHub 本身对仓库有 1GB 的软限制，单个文件不超过 100MB，注意定期清理老旧图片。

3. **URL 陷阱**  
   - 空格和特殊字符必须转义。
   - 如果用分支 `master`（旧项目），注意大小写，jsDelivr 区分大小写。
   - 自定义域名配置错误时，PicGo 可能生成 `raw.githubusercontent.com` 的链接，需要检查一遍。

4. **GitHub Token 权限**  
   Token 至少需要 `public_repo` 权限上传公共仓库。如果只勾选 `repo` 的某些子权限，可能报 404。另外 Token 要定期轮换。

5. **自动上传的并发问题**  
   如果你的 Agent 在短时间内生成大量图片并同时调用 GitHub API，可能触发 API 二级速率限制。建议实现一个简单的队列或者延迟上传，避免被断流。

## 可复用建议

- **命名规范**：统一用 `project/YYYY-MM-DD/hash-or-title.png` 格式，方便回溯和批量管理。
- **不用 `latest` 标签**：虽然 jsDelivr 支持 `@latest`，但除非你主动维护版本发布，不然它可能拿到缓存很旧的版本。
- **日志保留 commit sha**：每次上传后，在 Agent 的日志或数据库里记下原始图片对应 commit sha，方便以后迁移或清理。
- **监控仓库容量**：用 GitHub Actions 定时统计仓库体积，超过 500MB 就发通知清理。
- **结合 Cloudflare 进一步兜底**：可以自己域名 CNAME 到 jsDelivr，万一 jsDelivr 出问题，还能快速切换到其它源。

## 总结

GitHub + jsDelivr 搭建图床的方案算不上新鲜，但对于需要低成本、自动化、稳定运行图片链路的 Agent 和插件开发者来说，它仍然是非常务实的组合。你完全可以用不到 30 行代码就完成“生成图片 → 上传 → 返回 CDN 链接”的闭环，让报告里的图再也不会变成死链。

只要在设计上注意用 commit hash 固化版本、控制仓库大小、处理好 Token 权限，这套方案可以在很长一段时间内稳定服役，没有账单，也不需要维护服务器。

---

