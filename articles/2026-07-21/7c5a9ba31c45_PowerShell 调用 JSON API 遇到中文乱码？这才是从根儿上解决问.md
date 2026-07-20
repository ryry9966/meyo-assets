---
title: PowerShell 调用 JSON API 遇到中文乱码？这才是从根儿上解决问题的办法
feedId: 29847
source: 综合讨论
publishedAt: 2026-07-21
---

## 背景

在 Windows 上做 OpenClaw、Agent 或自动化插件开发时，经常需要用 PowerShell 调用第三方 JSON API 获取中文数据，然后交给下游处理。  
最常见的情况：写一个 `.ps1` 脚本，用 `Invoke-RestMethod` 拉回一段 JSON，结果中文全部变成 `????` 或 `锟斤拷`。保存到文件再读取，仍然乱码。更诡异的是，同一个脚本在 VS Code 里跑正常，在系统自带的 PowerShell 控制台里跑就坏。

这个问题困扰了大量工程化实践的开发者。很多人第一反应是“改系统语言”或者“换终端”，但根本原因其实在 **PowerShell 的默认输出编码与字节流处理方式** 上。

本文会直击本质，给出一个可复现的排障路径和工程化解决方案，让你以后再也不会被中文乱码绊倒。

---

## 问题根因：编码转换错位

### 现象拆解

假设我们调用一个返回中文内容的 API：

```powershell
$resp = Invoke-RestMethod -Uri 'https://api.example.com/data' -Method Get
$resp.data.title  # 输出变成 ???? 或者 Unicode 替换字符
```

`Invoke-RestMethod` 会将响应体自动转为 PowerShell 对象。如果 API 返回的 HTTP 响应头里 **没有声明 `charset=utf-8`**，或者声明了但 PowerShell 没有正确识别，它就会用系统的 ANSI 代码页（例如 Windows-1252 或 GBK）去解码 UTF-8 字节流。这就产生了典型的 **解码错配**。

在 PowerShell 5.1（Windows 自带的版本）中，`Invoke-RestMethod` 默认使用 ISO-8859-1 回退解码，中文自然就被“打坏”了。

---

## 标准做法：显式编码控制

### 1. 获取原始字节，自己解码

最稳妥的方式是放弃让 PowerShell 自动转换，直接拿到原始字节流，然后用 UTF-8 解码：

```powershell
$response = Invoke-WebRequest -Uri 'https://api.example.com/data' -Method Get
$rawBytes = $response.Content -as [byte[]]
$jsonString = [System.Text.Encoding]::UTF8.GetString($rawBytes)
$obj = $jsonString | ConvertFrom-Json
$obj.data.title  # 中文正常
```

这里故意用 `Invoke-WebRequest` 获取 `Content` 字符串，但把它转回字节是因为：即使 `Content` 已经被错误解码，`-as [byte[]]` 也能拿回原始字节（PowerShell 内部存储的是 ISO-8859-1 映射的字节数组）。然后用 `UTF8.GetString` 就能还原正确的中文。

> 踩坑点：如果 API 返回了 BOM 头（`U+FEFF`），`ConvertFrom-Json` 可能解析失败。可以先做一次 `TrimStart` 处理。

### 2. 强制要求正确的 Content-Type

如果你能控制 API 端，确保响应头包含 `Content-Type: application/json; charset=utf-8`。  
对于外部 API，可以在请求时添加 `-ContentType 'application/json; charset=utf-8'`，但这仅影响请求体解码，对响应解码无效。

真正有效的做法是使用 `Invoke-RestMethod` 的 `-ResponseHeadersVariable` 获取头，然后根据头信息做判断。不过最通用的还是上面的原始字节方式。

---

## 踩坑点集锦

### 乱码还会出现在哪里？

- **重定向到文件**：  
  `$jsonString | Out-File -FilePath out.json` 默认使用 Unicode (UTF-16LE) 编码，可能导致其他 UTF-8 工具读不出来。应加上 `-Encoding UTF8`。

- **控制台显示乱码**：  
  即使变量里存的是正确的中文，PowerShell 控制台渲染时也可能因为代码页是 936 (GBK) 而显示为问号。这是显示层问题，数据本身没坏。解决方法：  
  `[Console]::OutputEncoding = [System.Text.Encoding]::UTF8`，并确保终端字体支持中文（例如使用 Windows Terminal）。

- **跨版本差异**：  
  PowerShell 7+ (Core) 默认输出编码就是 UTF-8，很多问题自动消失。但如果你的部署环境是 Windows Server 2016 自带的 PowerShell 5.1，就必须手工处理。

---

## 工程化可复用方案

将这组编码防御逻辑封装成一个函数，以后所有 API 调用统一走它：

```powershell
function Invoke-JsonApi {
    param(
        [string]$Uri,
        [string]$Method = 'Get',
        [hashtable]$Headers = @{}
    )
    $response = Invoke-WebRequest -Uri $Uri -Method $Method -Headers $Headers -UseBasicParsing
    $raw = $response.Content -as [byte[]]
    if ($raw) {
        $json = [System.Text.Encoding]::UTF8.GetString($raw)
        # 去掉可能存在的 BOM
        if ($json[0] -eq [char]0xFEFF) { $json = $json.Substring(1) }
        return $json | ConvertFrom-Json
    }
    return $null
}
```

同时，在脚本最开头固定输出编码，避免后续管道传递时再次损坏：

```powershell
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
$OutputEncoding = [System.Text.Encoding]::UTF8
$PSDefaultParameterValues['*:Encoding'] = 'utf8'
```

这样一来，无论是 Out-File、Set-Content，还是直接打印到控制台，都不会出现二次乱码。

---

## 总结

中文 JSON API 调用的乱码，本质是 **字节流到字符串的解码过程使用了错误编码**。  
在 Windows PowerShell 5.1 中，默认的 ANSI 回退是罪魁祸首。通过手动接管字节解码、强制 UTF-8，并固定脚本运行环境的编码参数，可以从根源上消灭乱码。

- **根本解法**：拿原始字节 -> UTF8 解码 -> 转对象。
- **防御性编码**：脚本头部固定 `Console.OutputEncoding` 和 `$OutputEncoding`。
- **环境一致性**：条件允许时迁移到 PowerShell 7+ 或 Windows Terminal。

下次再遇到“PowerShell 把中文打坏了”，别再怀疑字体或系统语言，直接查编码链。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-21/080e2f9e836e2f83.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-21/458e28fde5192911.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-21/c11361dc6d564b36.png)

