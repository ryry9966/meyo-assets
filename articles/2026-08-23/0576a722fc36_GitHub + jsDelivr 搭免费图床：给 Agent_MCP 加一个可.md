---
title: GitHub + jsDelivr 搭免费图床：给 Agent/MCP 加一个可审计的图片出口
feedId: 34391
source: 综合讨论
publishedAt: 2026-08-23
---

## 背景

在 OpenClaw、Agent、MCP 这类自动化工作流里，图片经常不是给人看，而是给下游插件、Markdown 渲染器、知识库或截图产物使用。比如：Agent 生成的预览图、OCR 预处理图、插件封面、自动报告插图。

这些场景需要图片有稳定 URL，能被 HTTP 读取，最好还能脚本批量上传、可回滚、不绑信用卡。传统对象存储通常要实名、备案或付费；临时图床则可能过期、限流、夹广告，不适合长期自动化。

## 问题

如果只用本地文件，Agent 之间无法共享；如果硬编码临时图床，几个月后链接失效，知识库内容就废了。一个适合 OpenClaw/MCP 的图床，至少要满足：

- 可程序化上传，能封装成 MCP tool；
- 返回稳定、可访问的 URL；
- 有版本记录，方便审计；
- 不引入复杂后端。

## 选型

GitHub 公开仓库 + jsDelivr CDN 是一个够用的组合：

- GitHub 提供版本化存储和 Contents API，适合脚本上传；
- jsDelivr 可免费加速 GitHub 公开仓库内容；
- 对 Agent/MCP 友好：上传动作可以封装成工具，返回标准 URL。

需要提前认清限制：jsDelivr 只能加速公开仓库；GitHub API 有单文件大小限制；国内访问 jsDelivr 主域偶尔不稳定。

## 做法/步骤

### 1. 建仓

新建一个专门仓库，例如 `agent-images`，设为公开。不要和业务代码混放，避免二进制文件污染主仓库历史。

### 2. 生成最小权限 token

在 GitHub Settings → Developer settings → Fine-grained tokens 里，创建只对该仓库生效的 token，权限只勾选 Contents 读写。不要把账号主 token 写进脚本。

### 3. 上传脚本

用 Python 调用 GitHub Contents API。文件名建议使用内容哈希，避免同名覆盖：

```python
import base64, hashlib, os, requests

repo = "user/agent-images"
token = os.environ["GH_TOKEN"]
data = open("a.png", "rb").read()
digest = hashlib.sha1(data).hexdigest()[:12]
path = f"img/{digest}.png"

r = requests.put(
    f"https://api.github.com/repos/{repo}/contents/{path}",
    headers={"Authorization": f"Bearer {token}"},
    json={
        "message": f"upload {path}",
        "content": base64.b64encode(data).decode(),
    },
)
commit_sha = r.json()["commit"]["sha"][:7]
print(f"https://cdn.jsdelivr.net/gh/{repo}@master/{path}")
print(f"https://cdn.jsdelivr.net/gh/{repo}@{commit_sha}/{path}")
```

### 4. 封装 MCP 工具

将上述逻辑封装成 `upload_image` 工具：入参为本地路径或 base64，出参为 jsDelivr URL。OpenClaw/Agent 生成图片后可直接调用，后续 Markdown 渲染直接引用返回链接。

## 踩坑点

1. **私有仓库不可用**  
   jsDelivr 只服务 GitHub 公开仓库。如果图片含隐私、密钥、内部截图，这套方案不合适，应换 Cloudflare R2 或对象存储。

2. **国内访问波动**  
   `cdn.jsdelivr.net` 在国内可能被 DNS 污染或访问缓慢。可以维护备用域，如 `fastly.jsdelivr.net`、`gcore.jsdelivr.net`，但不要假设它们长期稳定。更稳妥的做法是在客户端做多域回退。

3. **缓存刷新慢**  
   jsDelivr 对同一 URL 缓存较久。覆盖同名文件可能不会立即生效。用内容哈希命名，或使用 commit SHA 固化版本，避免“更新了图但看到的还是旧图”。

4. **GitHub API 大小限制**  
   单文件 base64 后超过约 100MB 会失败；超过 10MB 建议压缩或改用 Git LFS。批量上传时注意 API rate limit。

5. **Token 泄露**  
   PAT 不要写进 MCP 配置前端、日志、报错信息或截图。建议环境变量注入，定期轮换，权限最小化。

6. **仓库膨胀**  
   二进制文件不断提交会使仓库历史变大。使用独立仓库可以隔离成本，必要时可以重建仓库或定期清理历史。

## 可复用建议

- 文件名使用 sha1/md5 内容摘要，天然去重、避免覆盖；
- commit message 记录来源模块和时间，便于回溯；
- 工具返回两个 URL：默认分支 URL 用于展示，commit SHA URL 用于固化版本；
- Agent 流程中增加重试：主域不可达时自动切换备用域；
- 非公开图片不要进入公开仓库。

## 总结

GitHub + jsDelivr 免费图床适合小规模、非敏感、自动化图片出口。它把“上传”变成一个可审计的 Git 操作，给 OpenClaw/Agent/MCP 提供稳定、可复现的图片 URL，同时避免引入对象存储复杂度。

它不是生产级 CDN，不适合大流量、热更新频繁或隐私图片。工程化的关键是：内容哈希命名、最小权限 token、多 CDN 回退，并清楚公开仓库边界。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/755e80dac3ab01b3.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/a9873da78a4bd945.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/9ca9f08c95247aab.png)

