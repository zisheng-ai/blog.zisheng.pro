---
title: 把 Agent Skill 当作架构边界：平台开发与发行版开发为什么要拆开
date: 2026-08-23 01:05:00
description: Agent Skill 不只是提示词集合，它会影响 Agent 从哪里读上下文、能修改什么、如何判断交付完成。本文以平台工程与 Browser Distribution 为例，解释为什么要拆开平台开发和发行版开发 Skill，并给出命名、目录、依赖和验收边界的实践框架。
categories:
  - [AI]
tags:
  - Agent
  - Skill
  - AI Infra
  - Software Architecture
  - Codex
cover: /images/agent-skill-architecture-boundary.webp
---

![平台开发通过已发布制品连接到发行版开发](/images/agent-skill-architecture-boundary.webp)

我最近整理一个浏览器平台工程时，发现最容易让 Agent 做错的，不是命令太多，而是把两类工作塞进了同一个 Skill：一边维护平台源码、协议和构建能力，另一边又让它操作具体产品的 Profile、签名、发布和验收。

看上去只是目录里少了一个文件夹，实际却把所有权混在了一起。Agent 一旦拿到模糊的“发布浏览器”指令，很容易从平台源码直接构建、拿本地 package 去验证产品，或者把签名、CDN、更新通道当成平台默认能力。每一步单独都可能跑通，组合起来却会破坏交付边界。

## 一句话总结

Skill 应该按“谁拥有决策和证据”拆分，而不是按“这些命令能不能连着执行”拆分。平台开发与发行版开发之间，最重要的产物是已发布、精确版本锁定的 package，而不是共享一个工作目录。

## Skill 不是知识库目录

很多 Skill 最初都像一份团队 Wiki：构建命令、常见错误、发布步骤、文档链接全放进去。它在任务少时很方便，任务一多就会开始替 Agent 做隐式决策。

例如下面这两类问题，表面都和“浏览器构建”有关，实际输入完全不同：

| 工作 | 真正拥有者 | Agent 应读取的输入 | 交付证据 |
| --- | --- | --- | --- |
| 修改 Kernel、CLI、Build Kit、Runway Core | 平台团队 | 平台源码、基线、测试、package manifest | 测试、tarball、registry readback |
| 配置品牌、内置扩展、签名、发布通道 | 发行版团队 | Profile、lock、产品 workspace、受控凭据 | native artifact、签名回读、CDN、旧客户端升级 |

如果同一个 Skill 同时覆盖它们，Agent 很容易把“本地源码已经通过测试”说成“发行版已经采用”，或者让发行版直接依赖平台的 sibling checkout。前者缺少发布和安装证据，后者会让本地环境掩盖真实的 package 边界。

## 我会把边界写进名字和目录

拆分不是把原文复制两份，而是让名称先表达工作对象。

```text
.codex/skills/
  platform-develop/
    平台源码、Kernel、Build Kit、CLI、公开 package

skills/
  distribution-develop/
    产品 Profile、私有 workspace、签名、发布、客户端验收
```

平台 Skill 放在平台仓库的 `.codex/skills/`，含义是：它服务这个仓库的维护者，不应被产品安装、CI 或运行时依赖。发行版 Skill 则面向拥有产品配置和交付责任的工程。两者可以引用同一套平台契约，但不能共享“直接读取平台源码”这条捷径。

目录位置不是审美问题。它决定 Agent 在任务开始时优先加载什么上下文，也提醒维护者：这里的规则究竟是本地开发习惯，还是要跟着某个产品一起交付的操作契约。

## 把依赖方向写成 Agent 能检查的规则

我通常会要求平台与发行版满足三个约束。

第一，发行版只消费 registry 中已发布的精确 package。禁止 `file:`、`link:`、`workspace:`、本地 tarball、sibling lookup 或跨仓 symlink。这样能强迫每次采用都经过版本、锁文件和干净安装。

第二，产品专属内容留在产品侧。品牌、Bundle ID、私有 Extension、Runway 实现、签名密钥引用、发布渠道与真实客户端验收，都不是平台的默认输入。平台可以提供 CLI 和安全 primitive，但不应该猜测产品路径或替产品保存凭据。

第三，交付状态必须拆开记录。测试通过、package 已发布、产品已构建、签名完成、CDN 可读、更新通道已激活、旧客户端成功升级，分别是不同事实。Skill 的价值不在于让 Agent 多跑几条命令，而在于避免把前一个状态误报成后一个状态。

## 一个足够小的判断框架

当我准备新增或拆分一个 Skill 时，只问四个问题：

1. 这个任务最终由谁对用户结果负责？
2. Agent 为完成任务必须读取哪些 source of truth？
3. 哪些凭据、产物和环境只能由特定工程拥有？
4. 什么证据能证明交付，而不只是证明代码存在？

四个答案相同，通常可以留在一个 Skill 里。只要其中一个答案分叉，特别是出现“平台已发布 package”与“产品已签名 artifact”两种证据时，就应该拆开。

这套方法不只适用于浏览器。任何同时维护 SDK、平台服务和多个产品的团队，都可以用同样的方式收紧 Agent 的工作面：让 Agent 在正确的仓库、正确的依赖方向和正确的验收门槛里工作。这样获得的不是更多流程，而是更少的错误捷径。

## 要不要拆

如果一个 Skill 既能修改共享平台，又能替多个产品发布，值得拆。先按所有权拆，再用明确的 package、Profile 和验收证据连接。

如果工作只发生在一个产品仓库，且没有跨仓发布或受控凭据边界，暂时不必为了形式拆 Skill。真正需要关注的是：Agent 是否会因为错误的上下文和依赖方向，做出无法被交付证据支撑的结论。

> 本文使用 [writting-skill](https://github.com/zisheng-ai/writting-skill) 辅助写作。项目已开源，欢迎在 GitHub 点个 Star。
