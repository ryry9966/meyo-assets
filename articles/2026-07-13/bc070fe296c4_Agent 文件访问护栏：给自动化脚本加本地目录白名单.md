---
title: Agent 文件访问护栏：给自动化脚本加本地目录白名单
feedId: 28897
source: 综合讨论
publishedAt: 2026-07-13
---

## 背景：Agent 操作文件系统的隐形风险

在 OpenClaw 的生态里，越来越多的 Agent 通过 MCP 服务器或直接调用本地工具来读写文件：下载邮件附件、导出报表、修改配置文件。这类操作一旦放开，Agent 就可能删除 .ssh 密钥、覆盖 /etc/hosts，甚至读取 /proc 环境变量。即便不涉及恶意攻击，一个错误的 Prompt 也足以让自动化流程“误伤”关键目录。

“目录白名单”是最小权限原则在文件系统上的直接落地：只允许 Agent 访问预先指定的目录（及其子路径），其他读写一律拒绝。这个想法简单，但落到工程实现时，有很多细节容易踩坑。下面就以一个典型场景——Node.js/TypeScript 实现的 MCP 工具或 OpenClaw 自定义插件——来拆解一套可复用的本地目录白名单实现。

## 问题定义：你需要限制的不只是“路径开头”

在 Agent 调用工具前加一层校验，很多人很快写出类似 `if (!targetPath.startsWith(allowedDir))` 的判断。但这存在几个典型漏洞：

1. **目录穿越**：`/allowed/../etc/passwd` 通过 `startsWith` 检查，但实际解析后指向了 `/etc/passwd`。
2. **符号链接逃逸**：白名单目录下某个子目录是个符号链接指向 `/var/run`，绕过限制。
3. **尾部匹配欺骗**：`/allowed-fake/secrets` 也能通过 `startsWith('/allowed')`。
4. **大小写与规范化差异**：macOS / Windows 文件系统大小写不敏感，但字符串比较大小写敏感。

因此，真正的“目录白名单”校验需要同时完成：路径解析、真实路径验证、且保证规范化比较。

## 做法：构建一个可复用的文件访问网关

下面以 Node.js 环境为例（Python `pathlib.resolve` + `PurePosixPath` 思路等同），实现一个 `PathGate` 工具模块，用在 MCP 服务的文件操作入口。

### 第 1 步：定义白名单与配置规范

```ts
// pathGate.ts
import { resolve, normalize, sep } from 'path';
import { realpathSync, symlinkSync } from 'fs';

interface PathGateConfig {
  /** 允许读写的白名单目录（绝对路径），必须规范化 */
  allowedDirs: string[];
  /** 是否在运行时实时解析符号链接（默认 true） */
  resolveSymlinks?: boolean;
  /** 是否拒绝目录穿越（默认 true） */
  blockTraversal?: boolean;
}
```

### 第 2 步：实现“净化解密”函数

核心逻辑：先规范化用户传入的相对路径或含 `..` 的路径，再解析符号链接得到真实路径，最后逐段比对白名单目录。

```ts
function sanitizeAndCheck(
  target: string,
  allowedDirs: string[],
  resolveSymlinks = true
): string {
  // 1. 初步规范化：解析相对路径，消除 .. 和 .
  let normalized = resolve(target);

  // 2. （可选）解析符号链接的真实路径，避免逃逸
  if (resolveSymlinks) {
    try {
      normalized = realpathSync(normalized);
    } catch (err) {
      // 文件可能尚未创建，可退而求其次：检查上级目录是否存在
      throw new Error(`Path resolution error: ${(err as Error).message}`);
    }
  }

  // 3. 再次规范化，保证平台一致性
  normalized = normalize(normalized);

  // 4. 严格白名单比对：使用分隔符补尾，避免部分匹配
  const allowed = allowedDirs.some(dir => {
    const normalizedDir = normalize(resolve(dir));
    // 确保 allowedDir 以路径分隔符结尾，防止 '/var/app' 匹配到 '/var/app2'
    const dirWithTrailing = normalizedDir.endsWith(sep)
      ? normalizedDir
      : normalizedDir + sep;
    return normalized.startsWith(dirWithTrailing) ||
           normalized === normalizedDir; // 允许直接等于目录本身
  });

  if (!allowed) {
    throw new Error(
      `Access denied: "${target}" resolves to "${normalized}" which is outside allowed directories.`
    );
  }

  return normalized;
}
```

### 第 3 步：集成到 MCP 工具或 Agent 插件

假设你有一个 MCP 服务器暴露 `read_text_file` 工具，只需在处理函数开头调用网关：

```ts
server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const path = sanitizeAndCheck(request.params.arguments.path, [
    '/home/user/agent-workspace',
    '/opt/data/exports'
  ]);
  // ... 继续读取文件
});
```

对于写操作、文件列表操作同理，建议对读/写分别设置白名单，进一步缩小权限。

## 踩坑点与排障记录

1. **`realpathSync` 对不存在文件抛错**：如果 Agent 要创建新文件，该文件尚不存在，`realpathSync` 会失败。处理策略：先对父目录解析真实路径，然后再拼接文件名，比较父目录是否在白名单内。
2. **Windows 盘符与 UNC 路径**：跨平台时，`resolve` 可能引入 `C:\` 盘符；白名单目录应使用 `path.win32.normalize` 保证一致性。
3. **性能开销**：每次调用都做 `realpathSync` 在高频操作时可能形成瓶颈。可以基于 LRU 缓存真实路径解析结果，但要注意符号链接更改后缓存失效的场景。
4. **容器 / chroot 环境**：如果 Agent 运行在 Docker 容器中，白名单可以用容器内路径定义，但要确保绑定挂载的路径不会逃逸。可结合 AppArmor 或 Seccomp 做内核层兜底。

## 可复用建议

- **抽象成可配置中间件**：不把白名单写死在每个工具里，而是作为通用文件访问层，所有文件操作都经过同一个 Gateway。
- **区分读与写白名单**：只读目录和读写目录往往不同，分别配置可降低风险。
- **提供默认拒绝的调用封装**：如果团队多人维护 Agent，把文件操作 API （如 `open`, `writeFile`）替换为带白名单检查的版本，让不安全的原始调用无法被直接使用。
- **记录审计日志**：被拦截的非法访问请求应该告警并记录，便于发现配置错误或潜在攻击。
- **测试用例覆盖**：用单元测试覆盖 `../../etc`、符号链接、尾部匹配、大小写变体等场景，避免回归。

## 总结

给 Agent 加目录白名单，本质是在自动化自由的“行动空间”上凿出一道防护栏。它不负责决策该不该访问，而是强制语言模型生成的任何文件操作都必须落在允许的目录范围内。这道护栏轻量、可嵌入现有 MCP 工具链，且与 OpenClaw 的插件机制天然契合。如果你正让 Agent 操作生产服务器上的文件，强烈建议在工具侧先把这道门关上，再想怎么开门。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-13/5f369f704ce9c07f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-13/72073819b726fbb9.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-13/d6371ef2deb478f7.png)

