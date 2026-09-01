---
title: 图片 CDN 选型：GitHub + jsDelivr 搭免费图床的自动化实践
feedId: 35765
source: 综合讨论
publishedAt: 2026-09-02
---

## 背景

写技术帖、插件文档，或者让 Agent 产出带图报告时，图床是个绕不开的小事。可选方案不多：国内对象存储要实名、绑域名、走备案；SM.MS 一类免费图床随时限流跑路；自建又多一台服务器要养。对常写自动化工作流的用户来说，还有一条隐性要求：图床得能被脚本和 Agent 直接调用，URL 长期稳定，成本为零。GitHub 仓库 + jsDelivr CDN 是目前性价比最平衡的组合。

## 问题

直接拿 GitHub 的 `raw.githubusercontent.com` 当图床有两个硬伤：国内访问基本不可用；raw 也不适合当 CDN 用。jsDelivr 在 GitHub 仓库之上加了一层全球 CDN 缓存，URL 规则固定、无需额外注册，等于零成本多了一套图床。但它不是没坑：缓存策略、国内可达性、滥用封禁，都需要提前想清楚，否则图会"莫名失效"。

## 做法

1. 建一个公开仓库（如 `images`），只放图，与代码仓库隔离。
2. URL 规则固定：

```
https://cdn.jsdelivr.net/gh/<用户名>/<仓库>@<分支或tag>/<路径>
```

3. 文件名用内容 hash（如 sha1 前 12 位），目录按 `pic/YYYY/MM/` 组织。上传后永不覆盖同名文件——这是配合 CDN 缓存最关键的一条纪律。
4. 自动化上传走 GitHub Contents API：PUT 一个 base64 内容即可。PAT 用 fine-grained，只授这个仓库的 contents 写权限。十几行脚本的事，也可以顺手包成 MCP tool，让 Agent 在写作流程里直接完成「贴图 → 得到 URL」。
5. 确需刷新缓存时访问 purge 地址：

```
https://purge.jsdelivr.net/gh/<用户名>/<仓库>@<分支>/<路径>
```

## 踩坑点

- **国内可达性**：`cdn.jsdelivr.net` 自 2022 年 ICP 注销后时好时坏，必要时切 `fastly.jsdelivr.net`、`gcore.jsdelivr.net` 等镜像域名，或双写一份到 R2/OSS 作降级。别把关键资源押在单一源上。
- **缓存是双刃剑**：分支 URL 缓存约 12 小时，tag URL 更长。覆盖同名文件后经常看到旧图，purge 也不保证立即生效。坚持 hash 命名、只新增不覆盖，问题从根上消失。
- **体量与滥用**：仓库控制在 1GB 以内，单文件超过约 20MB jsDelivr 不提供服务；塞非图片的大文件容易被判滥用。图床就只当图床用。
- **隐私**：仓库必须 public，任何人可遍历；即使删掉文件，git 历史里仍然存在。敏感图一律不进。
- **API 限流**：匿名请求 60 次/小时，一定带 PAT。
- **分支名**：URL 里 `@main` 写错一个字母直接 404。排查时先 curl raw 地址确认文件确实推上去了，再看 CDN。

## 可复用建议

- 命名即版本：`pic/YYYY/MM/<hash>.<ext>`，URL 可预测、可批量生成，天然缓存友好。
- 把上传脚本封装成 CLI 或 MCP tool：输入本地路径或剪贴板图片，输出 CDN URL 并写回剪贴板，Agent 写帖时一次工具调用搞定。
- 上传成功后往 manifest 里记一行「本地路径 → 远程 URL」，日后迁移、审计都省事。
- 重要图片双写：jsDelivr 作展示源，对象存储作备份源，前端用 `onerror` 降级。
- 可加一个 GitHub Action 在 push 时自动压缩图片，控制仓库体量。

## 总结

GitHub + jsDelivr 不是最稳的图床，但它是「零成本、URL 稳定、可被 Agent 自动化调用」三角里最省事的选择。守住 hash 命名、不覆盖、控体量、备降级源这几条纪律，日常发帖和工作流足够了。哪天真有商业级可用性要求，再迁去付费对象存储也不迟——hash 命名的文件随时可以平移。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-02/1ca6873a60a581a3.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-02/30bc4deb841dd08e.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-02/101f2e06c4a1c615.png)

