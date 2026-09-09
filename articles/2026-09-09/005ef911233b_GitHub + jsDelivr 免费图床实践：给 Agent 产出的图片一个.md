---
title: GitHub + jsDelivr 免费图床实践：给 Agent 产出的图片一个稳定外链
feedId: 36716
source: 综合讨论
publishedAt: 2026-09-09
---

## 背景

跑 Agent 自动化久了，图片是最常见的副产品：MCP 截图工具留下的运行快照、生成式插件产出的图表、自动化流程画出来的报表曲线。这些东西最终都需要一个稳定的 URL——写进博客、贴进 README、回传给前端渲染，都绕不开图床。

## 问题

可选方案不少，但各有代价：对象存储要花钱（个人量级不划算）；公共免费图床有容量、外链审核和跑路风险；直接裸链 `raw.githubusercontent.com` 在国内访问不稳定，且没有 CDN 缓存。我们真正需要的是：**零成本、纯 API 可自动化、外链相对稳定、随时可迁移**。GitHub 仓库 + jsDelivr 的组合基本满足。

## 做法

1. **建一个公开仓库**。私有仓库 jsDelivr 不回源，必须是 public。
2. **上传后拼外链**，格式固定：

```text
https://cdn.jsdelivr.net/gh/<user>/<repo>@<分支或tag>/<路径>
```

用 `@main` 取最新内容，用 `@v1` 这类 tag 取不可变快照。
3. **自动化接入**用 GitHub Contents API，核心三步：读文件 → base64 → PUT：

```bash
curl -X PUT \
  -H "Authorization: Bearer $GH_TOKEN" \
  https://api.github.com/repos/me/pics/contents/2025/06/a1b2c3.png \
  -d '{"message":"upload","content":"<base64>"}'
```

4. 更进一步：把这段逻辑封成一个 **MCP tool 或 CLI**，输入本地路径、输出 CDN URL，Agent 一句“把这张图传图床”就完成闭环。
5. 文件名用 **日期 + 内容 hash**，天然去重，也避免同名覆盖导致的缓存混乱。

## 踩坑点

- **缓存不即时**：分支引用（`@main`）的 CDN 缓存可达 24 小时，更新图片后旧图可能还在。手动刷：`curl https://purge.jsdelivr.net/gh/me/pics@main/xx.png`。正式资产建议直接走 tag/release，外链永不漂移。
- **单文件限制**：实测 20MB 以上的文件 CDN 会拒绝服务，大图先压缩再传。
- **必须 public**：仓库一转私有，所有外链立刻失效。
- **文件名别带中文和空格**：URL 编码问题会让外链在某些环境挂掉。
- **API 限额**：匿名请求额度很低，自动化必须挂 token，且做好失败重试。
- **国内可访问性有波动**：jsDelivr 历史上出过解析问题。别把它当唯一副本。

## 可复用建议

- **外链是派生物**：原始图片永远留在自己的仓库和本地，CDN 只是一层可替换的视图。
- 维护一份 `index.json`，记录 `本地路径 → CDN URL` 的映射，将来整体迁移只改这一份数据。
- 上传函数收敛到一处（MCP tool / CLI 均可），换 CDN 时只改 URL 拼接逻辑。
- 量级控制在个人文档、博客、Agent 产出这个尺度。GitHub 的定位是代码托管，拿它当高流量生产图床属于滥用，别招麻烦。

## 总结

GitHub + jsDelivr 是个人开发者和小团队性价比很高的一档选择：零成本、全 API 驱动、天然版本化，和 Agent/自动化工作流衔接顺畅。代价是缓存延迟、容量上限和可访问性风险需要自己兜底。工程上只要守住“原始资产自持、外链可替换”这条原则，这套方案可以放心用很久。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/48d9fd336f265fe2.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/178ddb0e7941ad12.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/5a668c1c5d728aa7.png)

