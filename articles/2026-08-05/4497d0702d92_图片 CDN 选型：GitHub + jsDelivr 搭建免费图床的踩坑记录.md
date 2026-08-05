---
title: 图片 CDN 选型：GitHub + jsDelivr 搭建免费图床的踩坑记录
feedId: 31732
source: 综合讨论
publishedAt: 2026-08-05
---

## 背景

做 OpenClaw/Agent 自动化流程的同学，大概率都遇到过这类需求：把截图、流程结果、MCP 工具输出图存到一个稳定可外链的地方。对象存储要钱，公共图床限速、不稳定，还经常对 API 调用设门槛。我试了一圈，落地方案是 GitHub 仓库存图 + jsDelivr 做 CDN，零成本、接口透明、可脚本化。

## 问题

- Agent 生成的图片（如流程截图）需要贴进 Slack、飞书或 Markdown 里，直接塞 Base64 会把上下文占满。
- 免费图床对自动化支持差，要么没 API，要么限流严重。
- 自建 MinIO 要维护，对小规模自动化流程不划算。

## 做法

1. 新建 GitHub 仓库（必须 public），比如 `openclaw-assets`。
2. 图片按日期分目录存放：`2024/06/01/xxx.png`。
3. 外链地址格式：

```
https://cdn.jsdelivr.net/gh/用户名/仓库名@main/2024/06/01/xxx.png
```

4. 把上述流程封装成脚本或 MCP tool：接收本地路径 → 调 GitHub Contents API 上传 → 返回 CDN URL。核心逻辑大约 40 行：

```python
def upload_and_get_url(local_path, owner, repo, token):
    content = base64.b64encode(open(local_path, 'rb').read()).decode()
    path = datetime.now().strftime('%Y/%m/%d') + '/' + basename(local_path)
    # PUT /repos/{owner}/{repo}/contents/{path}
    # headers: Authorization: token {token}
    # 返回 f"https://cdn.jsdelivr.net/gh/{owner}/{repo}@main/{path}"
```

## 踩坑点

- **默认分支**：jsDelivr 默认按 `@main` 取名。仓库是 `master` 分支的话必须写 `@master`，否则直接 404。
- **文件大小**：单文件控制在 20MB 内，jsDelivr 上限 50MB，超过就挂。Agent 截图动辄 1MB+，上传前务必压缩。
- **删除不生效**：CDN 节点有缓存，文件删了 URL 还可能存活一段时间。需要强刷就去 jsDelivr 官网提交 purge 请求。
- **文件名不要用中文**：浏览器会自动转码，部分客户端（如微信内置浏览器）会失效。统一用 `YYYYMMDD-HHMMSS.png`。
- **仓库必须 public**：private 仓库的图 jsDelivr 拉不到，除非上鉴权 CDN，那就不叫免费图床了。

## 可复用建议

- 图片先压再传：用 Pillow 或 tinypng 压到 200KB 以内，既能提速，也能降低 jsDelivr 限流概率。
- 追求一致性就带 commit hash：URL 里把 `@main` 换成 `@{commit}`，文件发布即不可变，适合需要回溯的场景。
- 打 tag 发布：URL 用 `@v1.2.3` 版本化，出问题可整体回滚。
- 在 OpenClaw 里用：把上传逻辑包成 MCP server 的一个 tool，输入本地图片路径，返回 CDN URL，省去手动脚本，自动化链路更顺。

## 总结

GitHub + jsDelivr 适合中小规模、对延迟不敏感的自动化场景。价值是零成本、API 透明，缺点是受 GitHub 政策影响且国内访问不完全稳定。纪律性建议：入仓前压缩、总量控制在几百 MB 内、核心图片本地留备份。量大了该上 OSS 还是得上，但在此之前，这套方案足够务实好用。

---

