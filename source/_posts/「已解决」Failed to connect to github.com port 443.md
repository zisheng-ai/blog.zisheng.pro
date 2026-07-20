---
title: 「已解决」Failed to connect to github.com port 443
date: '2023-07-18T11:20:35.047326+08:00'
description: git push 或访问 GitHub 时出现 Failed to connect to github.com port 443，优先按 DNS、代理与网络出口逐层定位。不要再依赖易失效的 hosts IP。
categories:
  - 工具
  - 问题排查
tags:
  - GitHub
  - Git
  - 网络
  - 代理
  - 排障
---

执行 `git push` 或 `git clone` 时出现 `Failed to connect to github.com port 443`，表示连接还没到 GitHub 的认证阶段，就在 HTTPS 建连前失败了。过去常见的“写死 hosts IP”现在并不可靠：GitHub 的 CDN、证书校验和 IP 都可能变化，错误映射还会制造更难排的 TLS 问题。

## 一句话总结

按“DNS 解析 → 本机代理 → 网络出口 → Git 配置”的顺序排查。hosts 只应作为明确、临时且可回滚的诊断手段，不是长期方案。

## 先确认浏览器和 Git 是否同样失败

在终端执行：

```bash
curl -I https://github.com
git ls-remote https://github.com/git/git.git HEAD
```

如果浏览器能打开而 Git 失败，重点检查 Git 的代理配置；如果两者都失败，则更可能是 DNS、代理客户端或网络出口的问题。

## 检查 Git 是否保留了旧代理

```bash
git config --global --get http.proxy
git config --global --get https.proxy
```

代理客户端退出、端口变化后，Git 仍会尝试连接旧地址。确认不需要代理时，清掉旧配置：

```bash
git config --global --unset http.proxy
git config --global --unset https.proxy
```

如果需要使用本地代理，确保地址、协议和端口与客户端实际监听一致；同时检查 shell 环境变量中的 `HTTP_PROXY`、`HTTPS_PROXY`。

## 检查 DNS 和网络出口

```bash
nslookup github.com
curl -v https://github.com
```

`nslookup` 异常说明先处理 DNS；DNS 正常但 `curl -v` 卡在连接阶段，则检查代理模式、防火墙或公司网络策略。公司网络下应优先使用合规的网络出口或咨询网络管理员，不要把公共 IP 固化进 hosts。

## 一个更稳定的替代：SSH

HTTPS 网络策略不稳定、且账户已配置 SSH key 时，可以将远程地址切到 SSH：

```bash
git remote set-url origin git@github.com:OWNER/REPO.git
ssh -T git@github.com
```

这不会绕过网络限制，但能避免 HTTPS 凭据与代理配置的额外变量。修改前可先执行 `git remote -v` 记录当前地址。

## 我的判断框架

优先修复：代理配置、DNS 和受控网络出口，它们才是可维护的根因。

避免采用：从第三方网站抄 IP 后长期写 hosts。它可能短暂恢复访问，却会在下一次基础设施变更时再次失效。
