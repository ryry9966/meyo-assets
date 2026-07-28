---
title: Agent 文件访问护栏：给自动化脚本加本地目录白名单
feedId: 30837
source: 综合讨论
publishedAt: 2026-07-29
---

# Agent 文件访问护栏：给自动化脚本加本地目录白名单

## 为什么需要文件访问护栏

在 Agent 工作流中，文件系统通常是最直接但又最危险的能力之一。不论是基于 OpenClaw 的自动化流程，还是自建 MCP 工具链，一旦 Agent 获得了 `read_file` / `write_file` 这类能力，它就可能意外或恶意地读写任意路径——配置文件、密钥、系统文件，甚至通过符号链接跳出预设工作区。

多数本地 Agent 的运行环境并没有默认的沙箱，仅仅依靠 Prompt 层面约束“只读某个目录”是完全不可靠的。真正可控的做法是在工具实现层设置文件访问护栏：只允许 Agent 读写明确指定的本地目录树，其余路径一律拒绝。

本文以在 OpenClaw 生态中构建一个带白名单目录检查的文件系统工具为例，梳理完整的实现路径与常见陷阱。

## 问题拆解

目标很明确：提供一套 `read_file` / `write_file` 工具，它们的行为与常规文件操作一致，但强制要求目标路径必须位于我们在配置中指定的一个或多个目录内。即使 Agent 构造了 `../../.env` 或 `/etc/passwd` 这样的路径，工具也必须直接拒绝执行。

核心技术点在于：

1. 路径规范化与解析
2. 白名单目录规范化
3. 防止绕过（符号链接、大小写、短路径等）
4. 在 OpenClaw / MCP 中如何传入白名单配置

## 实现步骤

### 1. 设计工具函数签名

假设我们要在 OpenClaw 自定义工具或 MCP Server 中提供两个能力：

- `read_file(relative_or_absolute_path: string)`
- `write_file(relative_or_absolute_path: string, content: string)`

白名单目录从环境变量 `AGENT_ALLOWED_DIRS` 获取，以逗号分隔，例如：
```
AGENT_ALLOWED_DIRS=/home/user/agent-workspace,/tmp/sandbox
```

### 2. 路径安全校验函数

下面是一个类型安全的 Node.js/TypeScript 实现，可直接用于 OpenClaw 自定义工具或 MCP Server 中：

```typescript
import * as path from 'path';
import * as fs from 'fs';

function getAllowedDirs(): string[] {
  const raw = process.env.AGENT_ALLOWED_DIRS;
  if (!raw) throw new Error('AGENT_ALLOWED_DIRS is not set');
  return raw.split(',').map(d => path.resolve(d.trim()));
}

function isPathAllowed(targetPath: string): boolean {
  const normalizedTarget = path.resolve(targetPath);
  const realTarget = fs.existsSync(normalizedTarget)
    ? fs.realpathSync(normalizedTarget)
    : normalizedTarget; // 文件尚不存在时仍按预期路径校验

  return getAllowedDirs().some(allowed =>
    realTarget.startsWith(allowed + path.sep) || realTarget === allowed
  );
}
```

**关键设计说明**：

- `path.resolve()` 消除 `..` 和相对引用。
- `fs.realpathSync()` 解析符号链接的真实路径，防止通过软链接跳出白名单。
- 当写文件时目标尚不存在，我们不能调用 `realpathSync`（会抛错），直接使用 `path.resolve` 得到的规范化预期路径来比对，同时要确保**中间目录**不包含符号链接欺骗。更严格的做法是对路径逐段向上检查存在部分，每一段都 `realpath`，但复杂度较高。工程上折中方案是对预期父目录 `realpath` 后做前缀判断，代码略。

### 3. 封装工具函数

```typescript
function safeReadFile(filePath: string): string {
  if (!isPathAllowed(filePath)) {
    throw new Error(`Access denied: ${filePath} is outside allowed directories`);
  }
  return fs.readFileSync(filePath, 'utf-8');
}

function safeWriteFile(filePath: string, content: string): void {
  if (!isPathAllowed(filePath)) {
    throw new Error(`Access denied: ${filePath} is outside allowed directories`);
  }
  fs.writeFileSync(filePath, content, 'utf-8');
}
```

### 4. 集成到 OpenClaw 工具中

在 OpenClaw 中注册自定义工具，直接在执行函数中调用 `safeReadFile` / `safeWriteFile` 即可。如果是通过 MCP Server 提供文件能力，则将这些函数挂载到 MCP tool handler 中。白名单目录通过环境变量注入，确保云函数或本地容器都能灵活配置。

## 踩坑记录

### 坑 1：Windows 盘符和路径分隔符

`path.resolve('C:\\allowed')` 与 `path.resolve('C:/allowed')` 在 Node 上结果一致，但 `startsWith` 比较时需注意尾部分隔符。添加 `path.sep` 后缀可避免 `/allowed-fake` 被误认为是 `/allowed` 的子路径。

### 坑 2：`fs.realpathSync` 性能与不存在文件

大量文件检查时频繁 `realpathSync` 可能带来 I/O 开销。作为 Agent 本地工具，这点开销可接受。更关键的是对**将要创建的文件**，父目录必须已存在，可单独对父目录 `realpath` 后做白名单前缀判断，而不是对不存在的文件调用 `realpathSync`。

### 坑 3：多个白名单目录的边界

假设白名单包含 `/workspace/project` 和 `/workspace/project-2`，`/workspace/project` 的 `startsWith(‘/workspace/project-2’)` 检查不会误伤，因为加了 `path.sep` 条件。但当目标路径正好等于某个白名单目录本身时，应允许访问，因此加上 `|| realTarget === allowed`。

### 坑 4：环境变量未设置时的行为

如果忘记设置 `AGENT_ALLOWED_DIRS`，工具应该直接明确报错，而不是无声地放开所有路径。在生产环境中，建议在 Agent 启动时进行健康检查，验证白名单目录是否存在且不为根目录 `/`。

## 可复用建议

1. **配置化白名单**：用环境变量或配置文件管理，避免硬编码，便于不同 Agent 实例复用。
2. **集中校验逻辑**：将所有文件访问工具都收口到同一个安全校验模块，避免遗漏新加的工具。
3. **增加审计日志**：当拒绝访问时，记录请求路径、Agent 任务 ID 和时间，便于后续发现异常行为。
4. **只读模式可选**：允许配置 `AGENT_READONLY=true`，即使路径在白名单内也只允许读，不允许写，用于更稳定的测试环境。
5. **测试覆盖**：编写单元测试覆盖各类边界：相对路径、符号链接、`..` 穿越、不存在的文件、非白名单路径，确保升级不会引入回归。

## 总结

给 Agent 的文件访问能力加白名单，并不是一次性的“写完就忘”的工作，而是需要在工具实现、配置管理和测试上持续投入。上面给出的实现虽然简短，但已经覆盖了路径遍历、符号链接等主要攻击面。在 OpenClaw 这类可扩展的 Agent 框架中，此类护栏能显著降低误操作或 Prompt 注入导致的文件泄露风险，是本地自动化实践的基础安全配置。

如果你的 Agent 已经开始接触本地文件，不妨花半小时为它加上这道围栏。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-29/0a06370b4353082d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-29/3bd92922a70acc4c.png)

