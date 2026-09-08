---
title: GitHub + jsDelivr 搭免费图床：一套对自动化友好的土办法
feedId: 36625
source: 综合讨论
publishedAt: 2026-09-08
---

## 背景

在 OpenClaw-CN 写教程、做演示，或者让 Agent 自动生成带截图的报告时，经常需要一个能外链、能热插的图片 URL。商业对象存储 + CDN 要花钱，国内加速还牵扯备案；GitHub raw 链接的 Content-Type 不稳定，直接当图床用体验很差。用 GitHub 公开仓库 + jsDelivr 的 `/gh/` 通道，是我目前实践下来成本最低、且整条链路可脚本化的方案。

## 问题

核心诉求：图片可直接外链、链接长期不失效、上传能从命令行或 MCP 工具一键完成、零成本。明确不追求的：高并发生产流量、私有图床。

## 做法

### 1. 建仓库，约定结构

新建一个公开仓库，比如 `openclaw-assets`，按 `img/yyyy-mm/` 分目录，文件名用内容 hash + 原扩展名。hash 命名既防重复上传，也天然规避缓存覆盖问题。

### 2. URL 格式

```
https://cdn.jsdelivr.net/gh/<user>/<repo>@<version>/<path>
```

- `@main` 指向分支，方便但有缓存延迟；
- `@<commit-sha>` 不可变，长期引用一律用它；
- 备用域名：`fastly.jsdelivr.net`、`testingcf.jsdelivr.net` 等，前端可做兜底切换。

### 3. 上传自动化（最值得做的部分）

GitHub Contents API 支持直接 base64 上传，不需要本地 clone，新增文件也不用先查 sha：

```bash
FILE="img/2025-06/3f2a9c.webp"
B64=$(base64 -w0 "$FILE")
curl -s -X PUT \
  -H "Authorization: Bearer $GITHUB_TOKEN" \
  "https://api.github.com/repos/$USER/openclaw-assets/contents/$FILE" \
  -d "{\"message\":\"upload $FILE\",\"content\":\"$B64\"}"
```

我把这段封装成了一个 MCP 工具：`img_upload(file) -> url`。Agent 截图后调用工具，直接输出可外链的 URL，整条链路无人工介入。

### 4. 缓存刷新

jsDelivr 边缘缓存比较长，如果覆盖同名文件，需手动请求 `https://purge.jsdelivr.net/gh/<user>/<repo>@<branch>/<path>` 触发刷新。所以更推荐"只新增、不覆盖"，hash 命名正好配合这一点。

## 踩坑点

1. **国内访问不稳**：`cdn.jsdelivr.net` 这几年在大陆时好时坏，别把它当核心资产的唯一来源。前端用 `onerror` 换备用域名，或接受偶尔加载慢。
2. **公开即公开**：push 上去的内容可能被搜索引擎和爬虫索引。自动化上传前务必过一道敏感信息检查，带 key、内网地址的截图绝不能进。
3. **GitHub ToS 灰色地带**：仓库主要用途如果是存放大量媒体文件，违反服务条款精神。个人博客、文档量级没问题，别把业务图床建在这上面。
4. **体积限制**：GitHub 单文件 50MB 警告、100MB 拒收，jsDelivr 对大文件也不友好。push 前先压一遍（webp/pngquant），单张控制在几百 KB。
5. **分支引用会过期**：`@main` 的引用在更新后可能长时间是旧缓存，要么 purge，要么一开始就用 commit-sha。

## 可复用建议

- 上传逻辑封装成 CLI 或 MCP 工具，Agent 工作流"生成即引用"；
- 维护一份 `manifest.json`（文件名 → URL、尺寸、日期），Agent 可检索已有素材，避免重复上传；
- 重要文档用 commit-sha 链接；将来迁移到真正的对象存储，只需换 URL 前缀；
- 定期跑个脚本巡检死链和仓库总大小。

## 总结

GitHub + jsDelivr 不是"生产级 CDN"，它是一个零成本、可脚本化、与自动化天然契合的个人图床。用它承载博客配图、文档插图、Agent 生成的素材绰绰有余；任何对可用性、合规性有要求的场景，老老实实上付费对象存储。先把上传工具链跑通，这套方案的价值才真正兑现。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-08/b3dc4dbf1caf0bab.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-08/7088f99436ab8d9d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-08/5199cb786e0edc4d.png)

