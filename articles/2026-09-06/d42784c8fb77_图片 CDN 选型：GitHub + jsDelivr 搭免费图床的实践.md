---
title: 图片 CDN 选型：GitHub + jsDelivr 搭免费图床的实践
feedId: 36318
source: 综合讨论
publishedAt: 2026-09-06
---

## 背景

在 OpenClaw 里跑 Agent 和 MCP 插件时，经常要产出图片：截图、图表、二维码、生成式插画。本地路径没法外链，发博客、写 README、给协作者看，都需要一个稳定的公网 URL。公共免费图床要么限流防盗链，要么不知哪天跑路；自建对象存储又有持续成本。权衡之后我选了 GitHub 仓库 + jsDelivr，稳定用了一年多，记录一下。

## 问题

需求很朴素：① 免费且长期可用；② URL 稳定可外链；③ 能被自动化流程调用——Agent 生成图片后自动上传、自动拿到链接。市面上多数图床死在第 2、3 条上。

## 做法

1. 建一个公开仓库，比如 `assets`，专放图片，不和代码混。
2. 生成 fine-grained token，只授予该仓库 Contents 的读写权限。
3. 上传走 Contents API，图片 base64 后 PUT，文件名带时间戳 + 内容哈希：

```bash
IMG="shot.png"
NAME="$(date +%s)-$(md5sum $IMG | cut -c1-8).png"
B64=$(base64 -w0 "$IMG")
curl -s -X PUT -H "Authorization: Bearer $GH_TOKEN" \
  https://api.github.com/repos/USER/assets/contents/images/$NAME \
  -d "{\"message\":\"upload\",\"content\":\"$B64\",\"branch\":\"main\"}"
echo "https://cdn.jsdelivr.net/gh/USER/assets@main/images/$NAME"
```

4. 外链走 jsDelivr 的 `/gh/用户/仓库@分支/路径` 格式。
5. 把第 3 步封装成几十行的脚本或 MCP 工具（如 `image.put`），入参本地路径、出参 CDN URL，Agent 流程里直接调用。

## 踩坑

- **缓存**：jsDelivr 对同名文件缓存很狠（分支路径约 12 小时），覆盖上传不生效。解法是哈希命名，文件名永不重复，把缓存问题变成非问题。
- **仓库体量**：GitHub 不鼓励当纯存储，单文件 100MB 硬限制，仓库建议控制在 1–2GB。截图先压缩，WebP 值得开。
- **并发冲突**：Contents API 并发写会报 409（SHA 冲突），多个 Agent 同时上传要加重试或串行队列。
- **隐私**：公开仓库意味着内容可被枚举。EXIF 记得清，敏感截图别走这条链路；token 最小权限，且不要被打进 Agent 日志。
- **国内访问**：jsDelivr 在大陆的可用性历史上波动过，面向国内读者的关键图片先实测，必要时留 raw 兜底或自建一层。

## 可复用建议

- 命名规范即缓存策略：日期 + 哈希，一劳永逸。
- 需要"不可变版本"就打 tag，用 `@v1.2.0` 引用比 `@main` 更稳。
- 封装成 MCP 工具是性价比最高的做法，一次写完，所有工作流受益。
- 加个简单的巡检脚本，定期 `curl -I` 校验关键 URL 返回 200。

## 总结

这套方案的本质是"GitHub 当 git 存储，jsDelivr 当分发层"：零成本、可全自动化、URL 长期有效，适合个人博客、文档、Agent 输出物这类中小流量场景。高并发商业分发请老实上对象存储 + 专业 CDN。在 OpenClaw 生态里，把它包成一个 MCP 工具后，"生图 → 上传 → 回链"可以完全无人值守，这是它最大的工程价值。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/113a95c187161122.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/7d513ef6bf8801a6.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/ac2a069059c20f9d.png)

