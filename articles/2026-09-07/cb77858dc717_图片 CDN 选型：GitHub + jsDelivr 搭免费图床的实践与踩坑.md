---
title: 图片 CDN 选型：GitHub + jsDelivr 搭免费图床的实践与踩坑
feedId: 36505
source: 综合讨论
publishedAt: 2026-09-07
---

## 背景

在 OpenClaw 的日常实践里，Agent 产出图很常见：数据图表、页面截图、生成的插画、运维报告插图。这些图片需要一个可公开访问的稳定 URL，才能塞进 Markdown 报告、Webhook 通知或前端页面。免费图床要么限速要么跑路，对象存储要实名、计费、写 SDK。最后我选了最"土"但最可控的组合：GitHub 公开仓库做存储，jsDelivr 做全球 CDN。

## 问题

直接用 `raw.githubusercontent.com` 有三个硬伤：国内访问不稳定；没有边缘缓存；raw 域名有速率限制，Agent 批量产出时容易 429。而自动化场景的核心诉求是：**上传动作可被代码化，返回的 URL 长期有效**。

## 做法

1. 建一个公开仓库，如 `cdn-assets`，按 `YYYYMM/` 分目录。
2. 用 GitHub Contents API 上传，最小可用函数（Python 示意）：

```python
import base64, hashlib, httpx

def upload(img: bytes, name: str) -> str:
    sha = hashlib.sha1(img).hexdigest()[:8]
    path = f"202506/{sha}-{name}"              # 哈希前缀防重名
    r = httpx.put(
        f"https://api.github.com/repos/USER/cdn-assets/contents/{path}",
        headers={"Authorization": f"Bearer {PAT}"},
        json={"message": f"add {path}",
              "content": base64.b64encode(img).decode()})
    r.raise_for_status()
    return f"https://cdn.jsdelivr.net/gh/USER/cdn-assets@main/{path}"
```

3. 把函数封装成 OpenClaw 的 skill 或 MCP 工具，Agent 就能"给图 → 拿 URL"一步完成。
4. 引用格式 `https://cdn.jsdelivr.net/gh/<user>/<repo>@main/<path>`；需要强制刷新缓存时请求 `https://purge.jsdelivr.net/gh/...`。

## 踩坑点

- **重名覆盖是最大的坑**。jsDelivr 缓存很激进，同名覆盖后边缘节点可能长期返回旧图。固定用"内容哈希 + 原文件名"，靠幂等命名绕开缓存问题。
- **`@main` 的缓存约 12 小时**，打 purge 可即时刷新；要"永久不可变"就打 tag，用 `@v1.0` 这类版本引用。
- **PAT 权限收敛**。用 fine-grained token，只授这一个仓库的 contents 写权限。classic token 全局可写，泄露等于交出账号。
- **别当生产图床**。jsDelivr 定位是开源项目静态资源，2022 年经历过 ICP 失效导致国内间歇不可用。低频个人/Agent 场景没问题，业务关键路径别押注。
- **仓库体积**。单文件超 100MB 直接被 GitHub 拒收；图片进了 git 历史很难瘦回来，上传前 webp 化、控制在 1MB 内，仓库逼近 1GB 就该换方案。
- **文件名带中文或 `+` 会翻车**，jsDelivr 对特殊字符的 URL 编码处理不一致，统一转 ASCII。
- **匿名 API 只有 60 次/小时**，务必带 token（5000 次/小时），高频场景再加本地去重。

## 可复用建议

- 上传前算哈希，本地维护 `hash → url` 映射（SQLite 即可），同一张图重复产出直接返回已有 URL。
- 写个兜底：jsDelivr 超时就切 `fastly.jsdelivr.net` 或回退 raw 域名，URL 只换前缀，上层逻辑零改动。
- 公开仓库全网可见，截图带密钥、内网域名的先打码——把这条写进 skill 描述，让 Agent 强制执行。
- 量起来了就换实现：同样的上传接口指向 R2/COS，Agent 调用代码不动。

## 总结

GitHub + jsDelivr 不高级，但便宜、稳定、可自动化，恰好匹配 Agent 场景"低频、程序化、URL 永久"的诉求。真正的工程量在把"上传"封装成可靠工具：幂等命名、权限收敛、缓存策略、失败兜底——这四件事做好，它就能长期服役。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/89eb5b7262108ebc.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/e7cd976d1b60ebae.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/c0004d3adb46ee60.png)

