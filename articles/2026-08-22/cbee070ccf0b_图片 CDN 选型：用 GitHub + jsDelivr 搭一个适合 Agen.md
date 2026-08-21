---
title: 图片 CDN 选型：用 GitHub + jsDelivr 搭一个适合 Agent 工作流的免费图床
feedId: 34108
source: 综合讨论
publishedAt: 2026-08-22
---

# 图片 CDN 选型：用 GitHub + jsDelivr 搭一个适合 Agent 工作流的免费图床

## 背景
在 OpenClaw/Agent/MCP/插件这类自动化场景里，经常需要把运行中产生的图片变成可外部访问的 URL：截图、生成图、插件预览图、故障现场快照等。直接放在本地、聊天窗口或 issue 里都不稳定，不能作为结构化数据传给下游。对象存储 + CDN 虽然更正规，但开通、计费、权限配置对个人项目或原型来说有点重。GitHub 仓库 + jsDelivr 是一套成本很低、且适合用 API 自动化的图床组合。

## 问题
这个方案不是“把图片传到 GitHub 就行”，要解决好几点：上传如何自动化、token 权限如何最小化、CDN 缓存失效、文件大小限制、仓库公开带来的隐私风险，以及国内网络可达性波动。

## 做法/步骤
1. 建一个 public 仓库，比如 `assets`，只放图片，不混入代码。
2. 生成 fine-grained personal access token，scope 只选这个仓库，权限只给 Contents 的 Read and write。将 token 写入环境变量，不要写进脚本或提交 git。
3. 用一个短脚本通过 GitHub Contents API 上传图片，并在本地做压缩和不可变命名。
4. 引用 jsDelivr URL：`https://cdn.jsdelivr.net/gh/<user>/assets@main/<path>`，其中 `@main` 按你的默认分支替换。

上传脚本示例：

```python
import base64, hashlib, os, sys, time, requests

token = os.environ["GH_TOKEN"]
repo = os.environ["GH_REPO"]  # 例如 yourname/assets
path = sys.argv[1]
data = open(path, "rb").read()
name = f"img/{int(time.time())}-{hashlib.sha1(data).hexdigest()[:10]}{os.path.splitext(path)[1]}"
r = requests.put(
    f"https://api.github.com/repos/{repo}/contents/{name}",
    headers={"Authorization": f"Bearer {token}",
             "Accept": "application/vnd.github+json"},
    json={"message": f"upload {name}",
          "content": base64.b64encode(data).decode()},
)
r.raise_for_status()
print(f"https://cdn.jsdelivr.net/gh/{repo}@main/{name}")
```

这个脚本会把图片写到 `img/` 目录下，文件名带时间戳和内容哈希，避免覆盖。

## 踩坑点
- **CDN 缓存不即时**：jsDelivr 对 GitHub 文件有较长缓存，覆盖同名文件很可能长时间返回旧图。所以不要用固定名 `a.png` 反复覆盖，用内容哈希或时间戳保证文件不可变。
- **文件大小限制**：GitHub 单文件不能超过 100MB，jsDelivr 对 GitHub 源约 50MB。实际用图最好先压缩到 1-2MB 以内，转 WebP 或限制宽度，失败率会低很多。
- **仓库历史膨胀**：Contents API 每次上传都会产生一个 commit，长期大量上传会让仓库 git 历史迅速变大。这个方案只适合低频公开图片，高频大文件应考虑对象存储。
- **公开性**：public 仓库的图片任何人都能访问，不能传内部截图、带 token 的调试图、用户隐私数据。
- **网络波动**：jsDelivr 部分地区可达性不稳定，如果目标用户在国内，需要提前测试；必要时准备对象存储或 CDN 兜底。

## 可复用建议
- 上传前统一压缩：用 Pillow 或 ImageMagick 转 WebP、限制最大宽高。
- 使用内容哈希命名，解决缓存和去重。
- 把 token 放在环境变量或 secret 管理里，脚本本身不硬编码。
- 在 OpenClaw 中封装成 CLI 或 MCP tool，让 Agent 可以调用：输入本地图片路径，输出公网 URL。
- 保留 `manifest.json` 记录原始文件名、哈希、URL，便于后期回溯和清理。

## 总结
GitHub + jsDelivr 图床适合开发、原型、插件文档、内部分享等非关键场景。优点是零成本、API 友好、易于接入 Agent 工作流；缺点是公开、缓存不即时、有容量和频率限制。对 OpenClaw/Agent 用户，把它定位成“可编程的公共图片暂存区”，而不是生产 CDN，是比较合理的用法。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/8b0c2508c1915958.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/3e08c3a3a1ea91bd.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/4dc43a125ee6f849.png)

