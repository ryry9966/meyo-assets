---
title: Windows 中文 JSON API 调用：为什么 PowerShell 会把中文打坏，以及如何干净修好
feedId: 30598
source: 综合讨论
publishedAt: 2026-07-27
---

## 背景：自动化脚本里突然全是问号

最近我在 Windows 上写一个 MCP 插件的辅助脚本，需要用 PowerShell 调用本地 Agent 暴露的 REST API，拿回一段包含中文任务描述的 JSON。API 用 Python FastAPI 提供服务，`/tasks` 端点在浏览器和 curl 里返回一切正常：

```json
{"id":1,"title":"修复登录模块超时问题"}
```

但在 PowerShell 里用 `Invoke-RestMethod` 接到的却是：

```
修复登录模块超时????
```

更糟的是，把返回内容 `Out-File` 到日志文件，再用记事本打开——全是乱码，代理链路上游的 Agent 完全无法正确解析。排查一圈后发现，不是 API 的锅，也不是 JSON 格式的锅，而是 PowerShell 对响应编码的理解出了偏差。这种偏差在英文内容下完全隐形，一旦中文出场，立刻打坏。

## 问题本质：PowerShell 靠“猜”来解码 UTF-8

Windows 上广泛存在的 PowerShell 5.1（至今仍是很多自动化环境的默认终端）底层基于 .NET Framework。`Invoke-RestMethod` 和 `Invoke-WebRequest` 在接收 HTTP 响应时，会检查响应头的 `Content-Type` 里有没有 `charset` 字段。如果服务器返回：

```
Content-Type: application/json
```

没有 `charset=utf-8`，PowerShell 就会根据 .NET 的规则退回到一个“默认 ANSI 代码页”，在简体中文 Windows 上通常是 GBK（CP936），在某些系统或配置下甚至退回 ISO-8859-1（Windows-1252）。而你的 API 大概率在生成 JSON 时已经老老实实用 UTF-8 编码了字节流。**解码器与编码器不匹配**，中文就会变成问号、拼音混杂乱码，或者奇怪的符号组合。

这解释了为什么 curl 和浏览器没事：它们通常默认假定 JSON 是 UTF-8，或者遵循 RFC 7159 的强制 UTF-8 要求。

## 复现步骤：亲眼看到它坏掉

**环境**：Windows 10 22H2，PowerShell 5.1.19041.5007，目标 API 为本地 FastAPI（uvicorn，返回 `application/json`，不带 `charset`）。

1. 准备一个最小的本地 API（`app.py`）：
   ```python
   from fastapi import FastAPI
   app = FastAPI()

   @app.get("/task")
   def get_task():
       return {"status": "已完成"}
   ```
2. 启动：`uvicorn app:app --port 8000`
3. PowerShell 调用：
   ```powershell
   $resp = Invoke-RestMethod -Uri "http://127.0.0.1:8000/task"
   $resp.status
   ```
   输出：`已?成` 或 `å·²å®Œæˆ`

4. 用 `Invoke-WebRequest` 检查原始字节：
   ```powershell
   $raw = Invoke-WebRequest -Uri "http://127.0.0.1:8000/task"
   $raw.RawContentStream
   ```
   再用正确的编码读出：
   ```powershell
   $reader = [System.IO.StreamReader]::new($raw.RawContentStream, [System.Text.Encoding]::UTF8)
   $reader.ReadToEnd()
   ```
   得到正确的 `{"status":"已完成"}`。

至此定位清晰：字节流没问题，只是 `Invoke-RestMethod` 的自动解码走错了代码页。

## 工程上的三种可靠修法

### 方案一：用 Invoke-WebRequest 手动指定 UTF-8（推荐）
放弃 `Invoke-RestMethod` 的自动解析，改为手动控制字节到字符串的转换。

```powershell
$response = Invoke-WebRequest -Uri "http://127.0.0.1:8000/task"
$jsonContent = $response.Content
# 强制按 UTF-8 重新解释 Content 的原始字节
$utf8Bytes = [System.Text.Encoding]::Default.GetBytes($jsonContent)
$correctString = [System.Text.Encoding]::UTF8.GetString($utf8Bytes)
$task = $correctString | ConvertFrom-Json
```

**原理**：`Invoke-WebRequest` 的 `.Content` 属性已经是错误解码后的字符串，但我们可以反向获取它的原始字节（利用 `Default` 编码即 ANSI），再用 UTF-8 重建。这招在 PowerShell 5.1 上很实用，需要注意的是如果响应包含非 ASCII 且 ANSI 也不兼容的字节，会略有信息损失，推荐直接操作 `RawContentStream`（见复现步骤）。

### 方案二：从服务器侧加 charset（最治本）
修改 API 响应头，让 `Content-Type` 变成 `application/json; charset=utf-8`。对于 FastAPI，可以在中间件或路由层面统一处理：

```python
from starlette.middleware.base import BaseHTTPMiddleware

class UTF8JSONMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        response = await call_next(request)
        response.headers["Content-Type"] = "application/json; charset=utf-8"
        return response

app.add_middleware(UTF8JSONMiddleware)
```

其他语言/框架同理，只要在响应头显式声明 charset，PowerShell 就会老老实实按 UTF-8 解码，问题从根源消失。

### 方案三：迁移到 PowerShell 7+
PowerShell Core（7.x）基于 .NET (Core) 而非 .NET Framework，默认对 `application/json` 的处理更加符合现代标准：如果没有 charset，它通常默认假定 UTF-8。在 PowerShell 7 中直接运行 `Invoke-RestMethod` 通常不会出现该乱码。对于能升级环境的 Agent/自动化用户来说，这是最简单的路径。

## 踩坑记录与可复用建议

在排查过程中，我还踩了三个常见坑：

1. **`> / Out-File` 输出错上加错**  
   `Invoke-RestMethod` 结果乱码后，再用 `> log.txt` 重定向，会因 PowerShell 的 `>` 使用 UTF-16 LE 写文件，而记事本默认按 ANSI 打开，导致乱码进一步变形。**做日志落盘务必指定编码**：`$result | Out-File -Encoding utf8NoBOM log.txt`。

2. **管道传 JSON 给内部函数时字符串再次转换**  
   当把 PowerShell 对象存为变量再转 JSON（`ConvertTo-Json`）回传给上游时，要留意它默认使用 `Unicode` (UTF-16) 编码。如果上游期望 UTF-8，需用 `[System.Text.Encoding]::UTF8.GetBytes()` 显式转换。

3. **部分 API 返回 `Content-Type: text/json` 或不标准类型**  
   这会导致 PowerShell 按纯文本处理，甚至引起 `Invoke-RestMethod` 无法解析。统一 API 响应内容类型为 `application/json; charset=utf-8` 是最省心的策略。

**可复用性建议**：如果你的 OpenClaw 工作流或 Agent 工具链需要在 Windows 上频繁调用 JSON API，最好在团队基线的 PowerShell 脚本模板里写死 UTF-8 解码逻辑，或者强制 API 网关注入 charset 头。不要寄希望于“默认没问题”，中文环境的窗口非常小。

## 总结

PowerShell 把中文 JSON 打坏的根因，不是 Windows 不支持 UTF-8，而是 HTTP 响应头缺失 `charset` 时，.NET Framework 的编码退避机制选择了与数据流不匹配的 ANSI 代码页。修法有三条路：客户端强制用 UTF-8 解字节、服务端补全 charset、或升级到 PowerShell 7。实践中推荐服务端补全 + 客户端防御性编码的双重保障。下次遇到“问号堆成山”，别先去怀疑 JSON 库，先 hexdump 看一眼字节，再查一下你的 PowerShell 到底拿了什么编码去读它。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-27/3f450f57c18c5f13.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-27/8753cfd57d911122.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-27/e2d8269f440301f2.png)

