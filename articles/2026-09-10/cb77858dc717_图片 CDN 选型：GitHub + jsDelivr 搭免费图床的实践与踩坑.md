---
title: 图片 CDN 选型：GitHub + jsDelivr 搭免费图床的实践与踩坑
feedId: 36948
source: 综合讨论
publishedAt: 2026-09-10
---

## 背景

在 OpenClaw 社区发帖、写项目文档、或者让 agent 归档运行截图时，都绕不开一个问题：图片放哪。商业图床要么收费、要么有失效风险，自建对象存储又显得太重。GitHub 仓库 + jsDelivr 是我目前用下来成本最低、迁移最方便的组合，这里把做法和坑一次性说清楚。

## 问题

直接贴 `raw.githubusercontent.com` 链接有三个痛点：

1. 国内访问不稳定，且 raw 链接不带 CDN 缓存，重复加载慢；
2. 图片混在代码仓库里会把仓库撑大，clone 变得痛苦；
3. 手动上传 → 网页里找链接 → 改格式的流程太繁琐，agent 自动化工作流里尤其难接受。

## 做法

1. **建独立公开仓库**，比如 `image-assets`，只放图不放代码，按 `images/2025/06/` 的年月目录组织。
2. **上传**：本地 git push 即可；想做自动化就写十几行脚本调 GitHub Contents API（base64 编码图片，PUT 提交），顺手拼好外链返回。
3. **链接格式**两种选法：
   - 跟分支：`https://cdn.jsdelivr.net/gh/<user>/<repo>@main/images/a.png`
   - 锁 commit：`.../<repo>@a1b2c3d.../images/a.png`，内容不可变，缓存永远正确
4. **自动化落点**：把上传脚本包成 CLI 或 MCP tool，让 agent 在「截图 → 上传 → 返回 Markdown 链接」一步完成。这是这套方案在 OpenClaw 工作流里最实用的形态。
5. **清缓存**：覆盖同名文件后，去 `purge.jsdelivr.net` 提交路径，或者干脆换 commit hash。

## 踩坑点

- **国内访问波动**：jsDelivr 2022 年后国内 ICP 失效，直连时好时坏。重要场景必须有兜底，别把鸡蛋全放这一个篮子。
- **缓存陷阱**：`@main` 是有缓存的分支引用，覆盖同名文件后外链大概率还是旧图，要么 purge 要么用 commit hash。我默认推荐后者。
- **滥用判定**：仓库纯当图床且流量异常，可能被 GitHub 标记 abuse。仓库保持小而专，别传视频（单文件 100MB 硬限制）。
- **API 细节**：文件路径含特殊字符要编码；base64 上传大图容易超请求体限制，压缩后再传。

## 可复用建议

- 链接一律用 commit hash 拼接，分支名只做「最新版」预览用；
- 脚本返回值直接是 `![](链接)` 格式，贴进编辑器零加工；
- 脚本里做双写兜底：主链 jsDelivr，备份链指向 Cloudflare R2 或自己域名反代，切换只改前缀；
- agent 工作流里把「截图→上传→贴链接」封装成一个原子操作，减少中间态。

## 总结

这套方案的成本是零，本质就是一个 git 仓库，随时可以整体迁移，非常适合博客、社区帖、内部文档和 agent 产物的归档。但要清醒：它的短板是国内直连波动，面向国内 C 端用户的正式产品图，还是老老实实上对象存储。工具选型没有银弹，知道边界在哪，比吹它免费更重要。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/fdca8e4d3b87b1e3.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/b2a58d8558bf8e8c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/da12293476c91f2f.png)

