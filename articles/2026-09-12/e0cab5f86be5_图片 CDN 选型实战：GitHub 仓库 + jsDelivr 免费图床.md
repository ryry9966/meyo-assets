---
title: 图片 CDN 选型实战：GitHub 仓库 + jsDelivr 免费图床
feedId: 37186
source: 综合讨论
publishedAt: 2026-09-12
---

## 背景

做插件文档、Agent 产出截图、MCP 工具生成的架构图，都绕不开一个需求：拿到一个**可外链、长期有效**的图片 URL。对象存储要付费和签名逻辑，公共图床服务随时可能关停。GitHub 仓库 + jsDelivr 是老办法，但在自动化流程里要用好，得先把缓存语义和命名规则想清楚。

## 方案原理

本质是「GitHub 当存储，jsDelivr 当全球缓存层」。上传图片到公开仓库后，按规则拼 URL：

```text
https://cdn.jsdelivr.net/gh/<user>/<repo>@<分支或tag>/<path>
```

引用 `@main` 表示"最新"，引用 tag 则视为不可变版本。自动化上传用 GitHub Contents API 即可，一条 curl 进 pipeline：

```bash
curl -X PUT \
  -H "Authorization: Bearer $GH_TOKEN" \
  "https://api.github.com/repos/me/assets/contents/assets/2025/06/a1b2c3.png" \
  -d '{"message":"add img","content":"'$(base64 -w0 img.png)'"}'
```

返回 201 后拼 CDN URL，直接给 Agent 回填。

## 落地步骤

1. 建公开仓库，目录按 `assets/{yyyy}/{mm}/` 组织；
2. 上传前压缩（pngquant / mozjpeg），截图别直接扔 retina 原图；
3. 文件名用内容 hash 前 8 位，永不覆盖；
4. 封装成 MCP 工具 `upload_image`：压缩 → hash 命名 → API 上传 → 拼 URL 返回，Agent 截图后一步到位；
5. 稳定版素材打 tag/release，博客引用 tag 链接。

## 踩坑点

1. **缓存不刷新**：同名覆盖后 CDN 仍是旧图。解法就是 hash 命名；紧急情况用 `https://purge.jsdelivr.net/gh/...` 手动 purge。
2. **仓库体积**：单文件 100MB 硬限制，仓库建议控制在 1GB 内，它不是网盘。
3. **大陆可达性波动**：jsDelivr 历史上间歇性抽风，重要页面留 raw.githubusercontent 备链，或换 `fastly.jsdelivr.net` 前缀。
4. **公开仓库无隐私**：能拼 URL 就能访问，截图前检查 token、日志、客户信息。
5. **API 限流**：5000 次/小时个人够用，但批量迁移别在循环里无 sleep 打 API。

## 可复用建议

- 命名规范一次定死，链接一旦发出永不失效；
- 前缀设计成可替换，未来迁 R2/OSS 只改域名；
- 中低频外链场景（文档/博客/Agent 输出）放心用，高流量商业化场景直接上对象存储，别在这套方案上硬撑。

## 总结

这套方案零成本、免运维，核心代价是要**主动管理缓存语义**——hash 命名解决九成问题，剩下交给 purge 和 tag。把它当静态资源镜像而非动态存储，预期对了，就能稳定用很久。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/558aa7a9d77512b3.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/8eacb3cff37d6767.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/34c47e6fef6cc1ec.png)

