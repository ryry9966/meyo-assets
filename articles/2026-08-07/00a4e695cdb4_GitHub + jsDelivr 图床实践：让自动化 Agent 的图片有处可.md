---
title: GitHub + jsDelivr 图床实践：让自动化 Agent 的图片有处可栖
feedId: 31977
source: 综合讨论
publishedAt: 2026-08-07
---

## 背景：自动化流程里的“最后一公里”图片问题

在 OpenClaw、Agent、MCP 插件这类自动化实践中，图片生成已非常普遍——截图、数据图表、OCR 结果标注、渲染后的仪表盘卡片。但这些图片如何稳定地嵌入到通知、日志或二次分发的消息里，往往成了被忽略的一环。

本地方案不可行，容器重启即丢；挂载卷成本高且跨机器访问困难；第三方图床要么限流严重要么 API 不够透明。我需要的是一套**完全免费、可脚本化、链路可控的图片 CDN 方案**，并且能和 OpenClaw 的自动化行为自然结合。

## 问题拆解

一个适合自动化链路的图床方案需满足：

1. **通过代码完成上传**，而非手动拖拽；
2. **返回可直链的 URL**，能直接被 Markdown/HTML 使用；
3. **不依赖任何单一商家的隐性配额**，不存在随时被清退的风险；
4. **全球可用**，至少在国内开发环境与海外 VPS 上都能获取。

直觉上我会排斥“免费图床”这种字眼，因为大多伴随低效或安全风险。但 GitHub 仓库 + jsDelivr CDN 的组合在工程化维度上意外地平衡：所有图片即代码仓库中的二进制文件，CDN 只做分发，权限隔离依赖 GitHub 自身的 Token 体系，无额外服务耦合。

## 方案骨架：GitHub Repo 作存储，jsDelivr 作分发

核心逻辑极简单：

- 在 GitHub 创建一个公开仓库（或仅存放图片的专用 repo）；
- 通过 GitHub API 将图片以 base64 或直接二进制方式 push 到指定路径；
- 利用 jsDelivr 的 GitHub 加速规则，拼接出静态资源 URL。

jsDelivr 对于 GitHub 资源的 URL 模式为：

```
https://cdn.jsdelivr.net/gh/{user}/{repo}@{branch}/{path}
```

例如图片路径为 `images/2024/1120/demo.png`，分支为 `main`，那么 CDN 链接就是：

```
https://cdn.jsdelivr.net/gh/用户名/仓库名@main/images/2024/1120/demo.png
```

不需要额外配置任何构建步骤，也不需要开启 GitHub Pages。jsDelivr 内部会缓存 GitHub raw 内容并提供全球节点，可用性远超直接访问 raw.githubusercontent.com（后者在某些网络环境下常常超时）。

## 实践步骤：与 OpenClaw 无缝配合

我以一个典型场景为例：Agent 使用 matplotlib 生成一张分析图，需将图片自动上传并拿到可访问的 URL，插入后续 Markdown 报告中。

### 1. 准备专用仓库与 Token

创建公开仓库，例如 `openclaw-assets`。生成一个具有 repo 权限的 GitHub Personal Access Token（classic，勾选 `repo` scope）。Token 仅作为环境变量注入运行环境，不硬编码。

### 2. 编写上传脚本（Python）

```python
import requests
import base64
import os
from datetime import datetime

token = os.getenv("GH_TOKEN")
repo = "yourname/openclaw-assets"
branch = "main"
image_path = "/tmp/chart.png"

with open(image_path, "rb") as f:
    content = base64.b64encode(f.read()).decode()

date_str = datetime.now().strftime("%Y/%m%d")
remote_path = f"images/{date_str}/chart_{datetime.now().timestamp():.0f}.png"
url = f"https://api.github.com/repos/{repo}/contents/{remote_path}"

resp = requests.put(
    url,
    headers={
        "Authorization": f"token {token}",
        "Accept": "application/vnd.github.v3+json",
    },
    json={
        "message": f"upload {remote_path}",
        "content": content,
        "branch": branch,
    },
)

if resp.status_code in (200, 201):
    cdn_url = f"https://cdn.jsdelivr.net/gh/{repo}@{branch}/{remote_path}"
    print(cdn_url)
else:
    raise Exception(resp.json())
```

这个脚本可被 OpenClaw 的行为节点直接调用，只依赖 `requests`。如果 Agent 框架已经具备执行 Python 的能力，甚至可以把上述逻辑封装为一个 MCP 工具，输入图片本地路径，输出 CDN 地址。

### 3. 在 Agent 链路中消费 URL

MCP 工具的输出可以直接注入到消息模板中：

```
分析完成，查看分布图：
![分布图](https://cdn.jsdelivr.net/gh/yourname/openclaw-assets@main/images/2024/1120/chart_1710000000.png)
```

由于 URL 是确定性的，不需要等待第三方回调，也不需要处理上传后的异步状态。

## 踩坑点与应对策略

**坑 1：jsDelivr 缓存刷新不及时**

对同一路径的文件做覆盖上传后，CDN 会持续提供旧版本，甚至长达 24 小时。关键认知：**不要把 jsDelivr 当作实时更新通道**。解决办法是每次上传生成唯一文件名（如带时间戳），从而绕过缓存。删除旧文件可以通过后续清理脚本处理，但不影响即时访问。

**坑 2：GitHub API 的速率限制**

未认证的 API 限制 60 次/小时，已认证 5000 次/小时。合理使用即可，无需额外申请。如果单 Agent 实例上传频率极高，可考虑引入本地缓冲，攒批上传。但正常图表生成场景远达不到限流阈值。

**坑 3：大文件与单文件 100MB 限制**

GitHub 建议文件 ≤ 50MB，硬限制 100MB。对截图和分析图表来说完全足够。若有视频或大量高清图需求，不适合此方案。

**坑 4：部分运营商 DNS 污染使 jsDelivr 不可达**

在某些国内云环境中，`cdn.jsdelivr.net` 可能解析异常。可以镜像到自定义域名，或使用 `fastly.jsdelivr.net` 等替代域名。结合 OpenClaw 海外部署场景，通常不构成问题。

## 可复用建议：让图床成为基础设施

1. **标准化路径前缀**：所有上传路径统一为 `images/YYYY/MMDD/uuid.png`，便于后续分析和批量清理。
2. **封装为 MCP 工具或插件**：将上传函数暴露给 OpenClaw 生态，输入文件内容或路径，输出 CDN URL。内部自动处理 base64 编码、路径生成和错误重试。
3. **关联域名**：如果觉得 `cdn.jsdelivr.net` 不够短，可以配置自己的域名 CNAME 到 jsDelivr，但需注意 SSL 证书的配置成本。
4. **定期清理与成本为零的维护**：GitHub 仓库免费无容量上限（合理使用），只要不用 Actions 进行频繁构建，这几乎是零运维负担的图床。

## 总结

GitHub + jsDelivr 组合并不是银弹，但在**自动化工作流中处理图片的发布与引用**这一具体场景下，它展现出非常可贵的工程节制：没有新服务依赖，没有平台绑定，唯一的外部依赖是 GitHub API，而 GitHub 本身大概率已是现有基础设施。用一个仓库、一个 Token、一段不到 30 行的上传逻辑，就能终结“生成了图却不知道存在哪”的尴尬。对 OpenClaw 用户而言，这种方案把图片 CDN 变成了一条可脚本化的管线，正好匹配 Agent 自主行为的诉求——稳定、透明、可控。

---

