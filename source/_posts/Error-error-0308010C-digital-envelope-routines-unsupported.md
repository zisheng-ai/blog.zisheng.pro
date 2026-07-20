---
title: 「已解决」error:0308010C:digital envelope routines::unsupported
date: 2024-10-24 17:48:38
description: Node.js 17+ 升级 OpenSSL 3 后，旧版 Webpack 等构建工具可能触发 error:0308010C。本文说明根因、临时绕过方案，以及应当优先做的长期修复。
categories:
  - [前端, Node.js]
  - 问题排查
tags:
  - Node.js
  - OpenSSL
  - Webpack
  - 构建
  - 排障
---

`error:0308010C:digital envelope routines::unsupported` 常出现在把旧前端项目切到 Node.js 17 或更高版本之后。它看起来像业务代码报错，实际是构建工具请求了 OpenSSL 3 默认不再支持的旧算法。

## 一句话总结

`--openssl-legacy-provider` 只能救急。长期方案是升级 Webpack/相关 loader，或把项目固定在兼容的 Node LTS；不要把这个环境变量当作永久配置。

## 如何确认根因

如果堆栈里包含 `webpack/lib/util/createHash`、`ERR_OSSL_EVP_UNSUPPORTED`，且故障发生在 `dev` 或 `build` 初期，基本就是这个问题。常见组合是旧版 Webpack 4 与 Node.js 17+。

## 临时恢复构建

macOS / Linux：

```bash
export NODE_OPTIONS=--openssl-legacy-provider
pnpm build
```

Windows PowerShell：

```powershell
$env:NODE_OPTIONS='--openssl-legacy-provider'
pnpm build
```

也可以仅给某个 script 设置该变量；重点是把它视为短期兼容层，并记录清理期限。

## 优先的长期修复

1. 升级 Webpack、React Scripts 或框架脚手架到支持 OpenSSL 3 的版本。
2. 同时升级依赖它们的 plugin、loader，避免 lockfile 把旧依赖重新拉回来。
3. 对短期无法升级的遗留项目，使用项目约定的 Node LTS，并通过 `.nvmrc`、Volta 或 CI 固定版本。

例如：

```txt
# .nvmrc
16
```

需要注意：选择旧 LTS 是为了给升级争取窗口，不是长期停留在已停止维护的运行时。

## 不要混淆的场景

如果错误发生在运行时的加密、签名或 TLS 逻辑中，而不是构建阶段，不能简单套用 legacy provider。应先确认算法、证书或上游 SDK 是否需要迁移，否则可能降低安全性。

## 我的判断框架

可以临时用：旧项目本地开发或一次性构建恢复。

优先修：持续交付、生产镜像和即将升级 Node 的项目；把工具链升级纳入技术债计划。
