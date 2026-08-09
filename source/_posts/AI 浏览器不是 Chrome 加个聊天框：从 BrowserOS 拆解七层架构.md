---
title: AI 浏览器不是 Chrome 加个聊天框：从 BrowserOS 拆解七层架构
date: 2026-08-10 02:43:34
description: AI 浏览器不只是 Chromium 加一个聊天界面，而是浏览器内核、原生集成、Agent Runtime、语义工具、模型连接与工程交付系统的组合。本文以 BrowserOS 源码为线索，拆解七层架构，并讲清构建、发布和多产物更新链路。
categories:
  - [AI]
tags:
  - AI Browser
  - BrowserOS
  - Chromium
  - Agent
  - MCP
  - CDP
  - AI Infra
cover: /images/ai-browser-browseros-seven-layers.webp
---

![AI 浏览器七层架构：六层运行时与一条工程交付控制面](/images/ai-browser-browseros-seven-layers.webp)

*AI 浏览器的运行时负责理解和执行任务，工程交付控制面负责把整套系统变成可安装、可发布、可更新的产品。*

最近我在基于 [BrowserOS](https://github.com/browseros-ai/BrowserOS) 看 AI 浏览器的实现。最开始我也有一个很自然的理解：AI 浏览器大概就是 Chromium 加一个侧边栏，在里面调用大模型，再给 Extension 多开几个权限。

真正顺着源码往下拆以后，我发现这个理解只能解释最上面的一层。

一个完整的 AI 浏览器，至少同时包含浏览器内核、产品级原生集成、交互界面、Agent Runtime、浏览器工具协议、模型与外部连接，以及支撑多平台构建、签名、发布和更新的工程系统。它表面上是一个桌面应用，内部更接近一套被打包到本机运行的垂直 Agent 系统。

本文以 BrowserOS 仓库的 [`5ddb0f2d0`](https://github.com/browseros-ai/BrowserOS/tree/5ddb0f2d04e15a2760f68729c217efaa2fd7437a) commit 为源码快照。BrowserOS 后续会继续变化，固定 commit 能让文中的目录、代码和发布流程保持可复查。

## 一句话总结

**AI 浏览器 = Chromium 产品底座 + 原生集成 + Agent Runtime + Browser Tool Plane + Model / MCP 生态 + Delivery Control Plane。**

这里最关键的不是某一个模型，也不是聊天 UI，而是两条完整链路：

```text
自然语言任务
→ Agent 决策
→ 语义工具
→ 浏览器协议
→ 真实页面状态与副作用

源码与版本
→ 构建
→ 签名与打包
→ 发布候选
→ Feed / OTA
→ 已安装客户端
```

第一条决定“它能不能完成任务”，第二条决定“它能不能作为一个软件产品持续交付”。

## 先把几个容易混在一起的对象分开

分析 AI 浏览器之前，最好先把几个名词的边界钉住。

| 对象 | 它负责什么 | 它不是什么 |
| --- | --- | --- |
| Chromium | 渲染、网络、存储、Profile、进程与 Extension 基础 | Agent Runtime |
| Extension UI | New Tab、Side Panel、设置与用户交互 | 整个 AI 浏览器 |
| Agent Loop | 调用模型、处理 Tool Call、持续推进任务 | 浏览器自动化协议 |
| Browser Tool | 面向模型的语义化浏览器能力 | 原始 CDP 命令 |
| CDP | 控制和观察 Chromium 的底层协议 | MCP |
| MCP | 将工具以标准协议暴露给 Agent Client | 页面执行引擎 |
| Updater | 让客户端发现和安装已发布版本 | Build 或 Release |

特别是 CDP 和 MCP，经常被泡在一起说。

CDP 的全称是 Chrome DevTools Protocol，它能创建 Tab、导航、执行 JavaScript、读取 DOM、截屏和监听浏览器事件。MCP 解决的是另一件事：如何把 `navigate`、`snapshot`、`act` 这类工具，以 Agent 能发现和调用的标准接口暴露出去。

简单说：**CDP 更靠近浏览器，MCP 更靠近 Agent。**

## BrowserOS 的两个产品形态

BrowserOS 当前仓库同时包含两个产品，这一点从根目录的 [README](https://github.com/browseros-ai/BrowserOS/blob/5ddb0f2d04e15a2760f68729c217efaa2fd7437a/README.md) 就能看到：

- BrowserOS：给人使用的 AI 浏览器，Agent 出现在 New Tab 和 Side Panel 中；
- BrowserOS neo：给外部 Agent 使用的浏览器，通过 MCP 接入 Claude Code、Codex、Cursor 等 Agent Client。

它们共享 Chromium 和相当一部分浏览器能力，但入口不同。

```text
BrowserOS
人 → New Tab / Side Panel → 本地 Agent Runtime → 浏览器工具

BrowserOS neo
外部 Agent → MCP → Browser Runtime → 浏览器工具
```

这篇文章以 BrowserOS 的主链路为主，因为它能完整展示一个“人使用的 AI 浏览器”如何工作。BrowserOS neo 放到后面作为架构演进：同一套底层浏览器能力，不一定只能服务内置聊天界面，也可以成为外部 Agent 的 Browser Runtime。

## 第一层：Interaction Surface

第一层是用户和 Agent 能看见的入口。

BrowserOS 的 [`apps/app`](https://github.com/browseros-ai/BrowserOS/tree/5ddb0f2d04e15a2760f68729c217efaa2fd7437a/packages/browseros-agent/apps/app) 是一个使用 WXT 和 React 构建的 Extension。它提供 New Tab、Side Panel、设置、Onboarding 和 Chat UI。

从 [`wxt.config.ts`](https://github.com/browseros-ai/BrowserOS/blob/5ddb0f2d04e15a2760f68729c217efaa2fd7437a/packages/browseros-agent/apps/app/wxt.config.ts) 可以看到几个重要信号：

- `chrome_url_overrides.newtab` 把 New Tab 指向 `app.html`；
- `sidePanel`、`tabs`、`bookmarks`、`history` 等权限提供浏览器交互入口；
- `host_permissions` 允许 Extension 访问本机 Server；
- `browserOS` 是 BrowserOS 自己增加的私有 Extension 权限；
- `update_url` 指向独立的 Extension 更新清单。

这一层负责把任务、上下文和执行状态呈现给用户，但不要把它叫作“Agent”。UI 可以发起任务、订阅流式结果、展示 Tool Call，却不应该承担模型循环和底层浏览器状态管理。

同样，MCP Client 也属于 Interaction Surface。对 BrowserOS neo 来说，Claude Code 或 Codex 就是另一个交互入口，只不过输入来自外部 Agent，而不是浏览器里的输入框。

## 第二层：Local Agent Runtime

用户提交任务后，请求进入本机运行的 Agent Server。

BrowserOS 的 [`apps/server`](https://github.com/browseros-ai/BrowserOS/tree/5ddb0f2d04e15a2760f68729c217efaa2fd7437a/packages/browseros-agent/apps/server) 使用 Bun 和 Hono，提供 `/chat`、`/mcp` 等接口。仓库的 [Agent README](https://github.com/browseros-ai/BrowserOS/blob/5ddb0f2d04e15a2760f68729c217efaa2fd7437a/packages/browseros-agent/README.md) 给出了默认结构：Extension 或 MCP Client 通过 HTTP 与本地 Server 通信，Server 再通过 CDP 控制 Chromium。

[`api/server.ts`](https://github.com/browseros-ai/BrowserOS/blob/5ddb0f2d04e15a2760f68729c217efaa2fd7437a/packages/browseros-agent/apps/server/src/api/server.ts) 负责启动 Hono 应用和 Bun Server，[`ai-sdk-agent.ts`](https://github.com/browseros-ai/BrowserOS/blob/5ddb0f2d04e15a2760f68729c217efaa2fd7437a/packages/browseros-agent/apps/server/src/agent/ai-sdk-agent.ts) 则把模型、Browser Tools、Filesystem Tools 和外部 MCP Tools 组合成 Agent Loop。

这一层至少承担：

- Conversation 和 Session；
- Browser Context；
- Model Provider 选择；
- Prompt 与消息标准化；
- Tool 注册与分发；
- Context Window 和 Compaction；
- Chat Mode 的只读工具限制；
- 流式输出、错误处理和运行指标。

为什么要单独起一个本地 Server，而不是全部塞进 Extension Background？

我的理解是，独立进程给 Agent Runtime 提供了更稳定的执行边界。它可以拥有自己的数据库、文件系统访问、长任务、模型 SDK、MCP Client 和生命周期，也能被 Extension、CLI、MCP Client 等多个入口复用。

但“本地运行”不等于天然安全。本地 Server 仍然需要处理监听地址、来源校验、鉴权、Tool 权限和文件访问边界。Local-first 只是部署位置，不是安全结论。

## 第三层：Model 与外部能力

Agent Runtime 自己不会推理，它需要连接 Model Provider；只会操作网页也不够，它还可能要写 Notion、查 Linear、发邮件或处理本地文件。

BrowserOS 在 Agent Runtime 中把能力分成三类：

```text
Model
  负责理解任务、生成下一步 Tool Call

Browser Tools
  负责观察和操作当前浏览器

External MCP / Connect Apps
  负责访问浏览器之外的业务系统
```

[`provider-factory.ts`](https://github.com/browseros-ai/BrowserOS/blob/5ddb0f2d04e15a2760f68729c217efaa2fd7437a/packages/browseros-agent/apps/server/src/agent/provider-factory.ts) 创建具体的 Language Model，[`mcp-builder.ts`](https://github.com/browseros-ai/BrowserOS/blob/5ddb0f2d04e15a2760f68729c217efaa2fd7437a/packages/browseros-agent/apps/server/src/agent/mcp-builder.ts) 连接外部 MCP Server。

这层的架构价值在于可替换性。只要上层 Agent Runtime 和下层 Tool Contract 保持稳定，模型可以从云端 Provider 切到本地模型，外部连接也可以从一个 MCP Server 换到另一个，而不必重写 Chromium 控制层。

当然，可替换不代表结果等价。不同模型对 Tool Schema、长上下文、截图理解和多步规划的表现不同，因此 Provider 切换之后仍然需要重新评测任务成功率，而不是只验证 API 能通。

## 第四层：Browser Capability

这是 AI 浏览器最容易被低估、也是最有工程价值的一层。

如果直接把原始 CDP 暴露给模型，模型会面对大量方法、参数、Target、Frame、Node 和 Session 状态。它不仅浪费 Token，也很难稳定地完成“点击登录按钮”这种在人类看来很简单的任务。

BrowserOS 在这一层继续拆成三部分：

```text
browser-mcp
    语义工具：navigate、snapshot、act、screenshot、tabs……
                ↓
browser-core
    状态抽象：BrowserSession、Page、Window、Navigation、Input
                ↓
CDP
    底层执行：Target、Page、Runtime、DOM、Input、Network……
```

[`browser-mcp`](https://github.com/browseros-ai/BrowserOS/tree/5ddb0f2d04e15a2760f68729c217efaa2fd7437a/packages/browseros-agent/packages/browser-mcp) 定义 Agent 看得懂的 Browser Tools，[`browser-core`](https://github.com/browseros-ai/BrowserOS/tree/5ddb0f2d04e15a2760f68729c217efaa2fd7437a/packages/browseros-agent/packages/browser-core) 则把 CDP 包装成有状态的 Browser Session。

例如模型想点击页面元素时，不应该先猜 CSS Selector，再拼一串 CDP 命令。更稳定的方式是：

1. `snapshot` 返回对 Agent 友好的页面结构和元素引用；
2. 模型根据语义选择引用；
3. `act` 接收引用和动作；
4. Browser Core 将引用解析到真实页面元素；
5. CDP 执行输入事件；
6. Tool 返回新的页面状态或 Diff。

所以，LLM 没有“点击页面”。**它生成了一个点击意图，Browser Tool Plane 才负责把意图变成确定性的浏览器操作。**

这一层还应该维护几个重要约束：

- Page、Tab、Window 的稳定身份；
- 导航前后元素引用何时失效；
- Frame 和弹窗如何归属；
- Screenshot、DOM Snapshot 与当前页面是否来自同一时刻；
- Tool 的副作用、超时、重试和错误语义；
- 多 Agent 并行时，谁拥有哪个 Tab 或 Session。

模型能力会快速变化，但这些状态问题不会自己消失。对 AI 浏览器来说，真正值得长期积累的往往就是这层。

## 第五层：Product 与 Native Integration

如果 BrowserOS 只需要做一个网页助手，到第四层为止，用 Chrome Extension 加本地 Server 也可能实现大部分能力。

它选择 fork Chromium，是因为产品还需要控制 Extension 权限之外的浏览器生命周期。

BrowserOS 在 Chromium 中加入了 `chrome.browserOS.*` 私有 API。Extension 侧的 [`adapter.ts`](https://github.com/browseros-ai/BrowserOS/blob/5ddb0f2d04e15a2760f68729c217efaa2fd7437a/packages/browseros-agent/apps/app/lib/browseros/adapter.ts) 暴露了版本读取、Pref 读写、路径选择和指标记录等接口。

这一层可以包含：

- Browser 和 Sidecar Server 的启动、停止与端口配置；
- 私有 Extension API 与权限；
- 浏览器设置和产品 Pref；
- 首次启动、Importer、默认页面和产品入口；
- 内置 Extension、产品图标、名称和 Bundle Identity；
- OS 级文件、进程、签名和更新集成。

它相当于 Extension 世界与原生 Chromium 世界之间的一座桥。Extension 仍然适合快速构建 UI，但产品不再受限于标准 Extension API。

代价也很明显：一旦 fork Chromium，就要长期维护 Patch Stack、解决上游冲突、承担跨平台编译和安全更新压力。fork 不是“更高级”的默认答案，它只是用更高的维护成本换取更深的产品控制权。

## 第六层：Chromium Kernel

最底层仍然是 Chromium。

Blink 负责页面渲染，V8 执行 JavaScript，Network Stack 处理请求，Profile 保存 Cookie、密码、扩展和用户设置，多进程架构隔离 Browser、Renderer、GPU 等进程。BrowserOS 可以在上面加 Agent，但不能绕开这些基础设施。

这一层还决定了 AI 浏览器的一个重要特性：它操作的不是一个抽象网页，而是用户真实的浏览环境。

真实环境意味着已登录 Session、Cookie、本地扩展、下载、权限弹窗、多窗口和长期 Profile；也意味着一次错误点击可能产生真实副作用。相比临时启动的 Headless Browser，这种能力更接近日常工作，也更需要清晰的权限、确认和审计边界。

BrowserOS 的 Chromium 相关代码集中在 [`packages/browseros`](https://github.com/browseros-ai/BrowserOS/tree/5ddb0f2d04e15a2760f68729c217efaa2fd7437a/packages/browseros)，包括 Chromium 版本 Pin、Patches、替换文件、资源和 Python 构建系统。

## 第七层：Engineering Delivery Control Plane

前六层解释了 AI 浏览器怎样运行，第七层解释它怎样成为一个能交付的软件产品。

我把它叫作 Engineering Delivery Control Plane，因为它不在一次 Agent 请求的调用链上，却贯穿 Browser、Server、Extension、CLI 和所有平台产物。

这一层负责回答：

- 基于哪个 Chromium commit 构建？
- 哪些 Patch 按什么顺序应用？
- BrowserOS 与 BrowserOS neo 各自包含哪些资源？
- macOS、Windows、Linux 如何编译和签名？
- 多个平台怎样证明来自同一个 Release Candidate？
- Browser、Extension、Server 和 CLI 怎样独立更新？
- 哪些产物只是 Staged，哪些已经真正发布？

这也是为什么我不建议只从运行时代码理解 AI 浏览器。没有这层，前六层可以做 Demo，却很难稳定追上 Chromium 安全更新并持续交付给用户。

## 一个任务如何穿过六层运行时

现在用一个具体任务把调用链串起来：

> 总结当前页面，并把结论写入 Notion。

### 1. Interaction Surface 接收任务

用户在 Side Panel 输入指令。Extension 读取当前 Tab 和选中的上下文，向本机 `/chat` 发起请求。

### 2. Agent Runtime 创建执行上下文

Server 恢复或创建 Conversation，组装 Browser Context、Model Config、Message History 和可用 Tools。

### 3. Model 生成 Browser Tool Call

模型并不知道页面完整内容，于是请求 `snapshot`。Agent Loop 不执行网页操作，只负责校验并分发这个 Tool Call。

### 4. Browser Capability 读取真实页面

`browser-mcp` 调用 `browser-core`，后者通过 CDP 从 Chromium 获得页面状态，再返回对模型更紧凑的 Snapshot。

### 5. Agent Loop 继续推理

模型根据 Snapshot 生成总结。随后它决定调用 Notion 对应的外部 MCP Tool。

### 6. External Capability 产生业务副作用

Notion MCP 接收结构化参数，创建或更新页面，并把结果返回 Agent Runtime。

### 7. Interaction Surface 展示结果

Server 将执行状态和最终答案流式返回 Side Panel，用户看到任务结果。

完整路径可以压缩成：

```text
User
  ↓
New Tab / Side Panel
  ↓ HTTP
Local Agent Server
  ↓
Agent Loop ←→ LLM Provider
  ↓ Tool Call
Browser MCP / External MCP
  ↓
Browser Core
  ↓ CDP
Chromium
  ↓
Page State / Real Side Effect
```

这里存在一个重要边界：BrowserOS 源码能证明这些组件和调用关系存在，但不能仅凭静态源码证明每个网站、每个模型、每个任务都能可靠完成。任务成功率仍然需要基于真实任务集、Provider 和环境做运行时评测。

## BrowserOS 怎样构建

BrowserOS 没有把构建过程写成一条越来越长的 Shell Script，而是在 [`bos_build`](https://github.com/browseros-ai/BrowserOS/tree/5ddb0f2d04e15a2760f68729c217efaa2fd7437a/packages/browseros/bos_build) 中实现了一套 Python Build CLI。

它的心智模型是：

```text
preset + product + platform + arch + switches
                         ↓
                ordered build steps
```

其中：

- `preset` 区分 Release、Debug 等构建目标；
- `product` 区分 BrowserOS 与 BrowserOS neo；
- `platform` 由当前 Host 决定；
- `arch` 可以是 x64、arm64 或 universal；
- `switches` 控制 Clean、Provision、Resource Mode、Sign、Upload 等行为。

[`planner.py`](https://github.com/browseros-ai/BrowserOS/blob/5ddb0f2d04e15a2760f68729c217efaa2fd7437a/packages/browseros/bos_build/core/planner.py) 根据这些输入生成有序步骤。一次完整构建大致经过：

```text
锁定 Chromium Source
→ 准备或校验源码缓存
→ Clean / Git Sync / Configure
→ 构建或下载公共资源
→ 应用文件替换和 Patch Stack
→ 注入 Extension、Onboarding、Server
→ 编译 Chromium
→ 签名
→ 打包
→ 上传候选产物
```

这里要注意两个点。

第一，BrowserOS 不是在 Chromium 编译完成后换个 Logo。Patches、替换文件、私有 API 和内置资源会在编译前进入源码树，最后生成一个新的 Chromium 产品。

第二，`build` 只负责生成当前 Host 能生成的产物。macOS 机器不能替代 Windows 完成 Windows 原生签名，单台机器也不能自然证明所有平台产物来自同一个版本。因此 Build 成功不等于完整 Release 完成。

## Build、Release、Update 是三件事

这三个词经常被混用，但它们对应三个不同的状态转换。

| 阶段 | 输入 | 输出 | 核心问题 |
| --- | --- | --- | --- |
| Build | 源码、依赖、配置 | 二进制和安装包 | 能不能构建出来 |
| Release | 多平台产物、签名、Evidence | 可发布候选和正式版本 | 这些产物是否一致、可信、可分发 |
| Update | 已发布版本、Feed、客户端策略 | 已安装客户端升级 | 客户端如何发现并安全消费新版本 |

一个 Binary 在本地编译成功，只能证明 Build；上传到对象存储但没有切换 Feed，通常只是 Staged；客户端能从正式 Feed 发现并安装，才进入 Update 链路。

把这些状态分开，会少掉很多“明明发布了，为什么用户收不到”的问题。

## BrowserOS 怎样发布多平台版本

BrowserOS 的完整 Release 设计记录在 [`release-ci.md`](https://github.com/browseros-ai/BrowserOS/blob/5ddb0f2d04e15a2760f68729c217efaa2fd7437a/packages/browseros/bos_build/docs/release-ci.md) 和 [`release-browseros.yml`](https://github.com/browseros-ai/BrowserOS/blob/5ddb0f2d04e15a2760f68729c217efaa2fd7437a/.github/workflows/release-browseros.yml) 中。

主流程是：

```text
从 main 手动触发
→ 创建或恢复不可变 Candidate Commit / PR
→ 公共资源只构建一次
→ Linux x64
→ Windows x64 + Signing
→ macOS arm64 / x64 / universal + Signing / Notarization
→ 校验所有 Lane Evidence
→ 合并未发生变化的 Candidate PR
→ 创建 Draft GitHub Release
→ 生成 Appcast Preview
→ 人工检查并 Promotion
```

这里最关键的设计是 immutable candidate。

多平台编译通常需要不同 Runner，耗时也不同。如果每个 Runner 都在执行时读取最新 `main`，那么 Linux 开始编译后，`main` 可能又进入了新 commit；最终几个安装包虽然写着同一个版本号，内部源码却可能不一致。

BrowserOS 先冻结 Candidate SHA，再让公共资源和各平台 Lane 都绑定这个 SHA、Browser Version、Component Version 和资源摘要。Release Gate 只有在完整矩阵通过后才允许继续。

这不是为了让 CI 看起来复杂，而是在解决一个真实的 Supply Chain 问题：**同一个 Release 中的多个产物，如何证明它们属于同一次构建意图。**

另一个值得借鉴的边界是“CI 只 Stage，不自动 Promote”。

完整 Release Workflow 会准备 R2 产物、Draft GitHub Release 和 Appcast Preview，但正式下载别名和更新 Feed 仍然需要显式发布。因为 Promotion 会直接影响已安装用户，它应该是一次独立、可检查、可回滚的决定。

## Update 不是一个 Updater

AI 浏览器不是单一 Binary。BrowserOS 至少包含 Browser、Extension、Local Server、BrowserOS neo Server、Onboarding 和 CLI 等可独立演进的产物。

| 产物 | 版本与发布方式 | 客户端发现方式 |
| --- | --- | --- |
| Browser | 多平台签名安装包、版本化对象、Release Metadata | Browser Appcast、正式下载别名或平台安装流程 |
| Extension | 固定 Extension ID、签名 CRX | `update-manifest.xml` |
| Local Agent Server | 独立 Server Bundle 和版本 | 签名 OTA Appcast |
| BrowserOS neo Server | Rust Native Bundle、独立版本 | 独立 OTA Feed |
| Onboarding | 独立资源 Bundle | Browser Build 或资源发布链路消费 |
| CLI | 独立 Workflow、Tag 和安装清单 | CLI 安装或升级入口 |

从 [`bos_build/README.md`](https://github.com/browseros-ai/BrowserOS/blob/5ddb0f2d04e15a2760f68729c217efaa2fd7437a/packages/browseros/bos_build/README.md) 和 Release Workflows 可以看到，Extension、Server 与 Browser 并不强制共享一个版本号。

这很合理。一个 Extension UI 修复不应该要求重新编译整个 Chromium；Server Agent Loop 更新也不一定要重新发布 Browser 安装包。独立版本减少了发布成本，但同时引入兼容性问题。

因此，一个完整的多产物更新系统至少需要：

```text
Component Version
+ Compatibility Contract
+ Immutable Artifact
+ Signature / Checksum
+ Mutable Feed / Alias
+ Client Update Policy
+ Rollback Strategy
```

其中最容易被忽略的是 Compatibility Contract。Browser 内置的私有 API、Extension、Server 和 Browser Tools 如果独立升级，必须明确支持哪些相邻版本。否则每个组件都更新成功，组合在一起反而不能运行。

## 为什么需要 BrowserOS neo

BrowserOS neo 展示了这套分层的另一个价值：交互层可以替换，底层浏览器能力仍然复用。

BrowserOS 面向人，Agent UI 内置在浏览器中；BrowserOS neo 面向外部 Agent，核心入口是 MCP，并提供 Dashboard、并行 Session 和 Replay。当前代码分别位于 [`claw-server-rust`](https://github.com/browseros-ai/BrowserOS/tree/5ddb0f2d04e15a2760f68729c217efaa2fd7437a/packages/browseros-agent/apps/claw-server-rust) 和 [`claw-app`](https://github.com/browseros-ai/BrowserOS/tree/5ddb0f2d04e15a2760f68729c217efaa2fd7437a/packages/browseros-agent/apps/claw-app)。

这意味着“AI 浏览器”可以有两种产品角色：

```text
Browser for Humans
浏览器是主产品，Agent 是内置能力

Browser for Agents
Agent 是调用者，浏览器是有状态的执行环境
```

第二种形态很值得 Agent / AI Infra 方向关注。它不要求每个 Agent Framework 都重新实现 Playwright 封装、登录态迁移、Tab 管理和页面观察，而是把浏览器沉淀成稳定的 Capability Server。

## 什么情况下只做 Extension，什么情况下 fork Chromium

最后回到一个务实问题：做 AI 浏览器，是不是都应该 fork Chromium？

我的判断是，不应该。

### 优先做 Extension

如果产品主要是：

- 总结、翻译和问答；
- 读取当前 Tab 内容；
- 在 Extension 权限范围内填表和导航；
- 快速验证产品价值；
- 可以接受跟随 Chrome 的产品入口和更新机制。

那么 Extension + Local Server 通常是成本更低的选择。先验证用户是否真的需要 Agent 完成这些任务，再决定是否接管浏览器底座。

### 考虑 fork Chromium

当产品明确需要：

- 浏览器级私有 API；
- 原生管理 Sidecar Runtime；
- 控制 New Tab、Importer、Profile、进程和产品身份；
- 内置并固定关键 Extension；
- 拥有安装包、签名、更新渠道和默认行为；
- 在 Chromium 上游之下维护长期产品差异。

这时 fork 才有足够收益覆盖维护成本。

### 重点建设 Browser Capability Layer

不管最终选择 Extension、Chromium fork 还是独立 Browser Runtime，Browser Capability Layer 都值得尽早独立出来。

因为模型会换、UI 会改、MCP Client 会增加，但 Page、Tab、Frame、Navigation、Snapshot、Input 和 Side Effect 的语义需要长期稳定。只要这一层保持清晰，上层可以接内置 Agent，也可以接外部 Agent；下层可以继续使用 CDP，也可以逐步增加更原生的能力。

## 我的判断

拆完 BrowserOS 后，我对 AI 浏览器的判断变得更具体了。

聊天框决定用户从哪里开始，模型决定下一步想做什么，真正把任务落到网页上的，是 Browser Tool Plane；真正让产品能追上 Chromium 并交付给用户的，是 Delivery Control Plane。

所以 AI 浏览器的长期壁垒，很可能不在某个模型 API，也不在一个漂亮的 Side Panel，而在三件更慢的事情上：

1. 能否把复杂浏览器状态压缩成稳定、低 Token、可验证的 Tool Contract；
2. 能否约束登录态、文件、页面操作和外部系统中的真实副作用；
3. 能否持续完成 Chromium 升级、多平台签名、多产物兼容和安全更新。

如果只想验证一个页面助手，Extension 已经足够；如果要做浏览器级产品，就要接受 Chromium fork 带来的长期成本；如果目标是让不同 Agent 稳定使用真实浏览环境，那么最值得先建设的，是独立于模型和 UI 的 Browser Runtime。

AI 浏览器看起来像一个应用，真正需要经营的却是两套系统：一套让 Agent 在浏览器里可靠工作，一套让这套能力持续、安全地到达用户机器。

## 参考源码

本文全部源码引用固定在 BrowserOS `5ddb0f2d0`：

- [BrowserOS monorepo architecture](https://github.com/browseros-ai/BrowserOS/blob/5ddb0f2d04e15a2760f68729c217efaa2fd7437a/README.md)
- [BrowserOS Agent architecture](https://github.com/browseros-ai/BrowserOS/blob/5ddb0f2d04e15a2760f68729c217efaa2fd7437a/packages/browseros-agent/README.md)
- [BrowserOS Extension configuration](https://github.com/browseros-ai/BrowserOS/blob/5ddb0f2d04e15a2760f68729c217efaa2fd7437a/packages/browseros-agent/apps/app/wxt.config.ts)
- [BrowserOS native API adapter](https://github.com/browseros-ai/BrowserOS/blob/5ddb0f2d04e15a2760f68729c217efaa2fd7437a/packages/browseros-agent/apps/app/lib/browseros/adapter.ts)
- [BrowserOS local Agent Server](https://github.com/browseros-ai/BrowserOS/blob/5ddb0f2d04e15a2760f68729c217efaa2fd7437a/packages/browseros-agent/apps/server/src/api/server.ts)
- [BrowserOS Agent Loop](https://github.com/browseros-ai/BrowserOS/blob/5ddb0f2d04e15a2760f68729c217efaa2fd7437a/packages/browseros-agent/apps/server/src/agent/ai-sdk-agent.ts)
- [Browser Core](https://github.com/browseros-ai/BrowserOS/tree/5ddb0f2d04e15a2760f68729c217efaa2fd7437a/packages/browseros-agent/packages/browser-core)
- [Browser MCP](https://github.com/browseros-ai/BrowserOS/tree/5ddb0f2d04e15a2760f68729c217efaa2fd7437a/packages/browseros-agent/packages/browser-mcp)
- [Browser build system](https://github.com/browseros-ai/BrowserOS/blob/5ddb0f2d04e15a2760f68729c217efaa2fd7437a/packages/browseros/bos_build/README.md)
- [Browser release CI](https://github.com/browseros-ai/BrowserOS/blob/5ddb0f2d04e15a2760f68729c217efaa2fd7437a/packages/browseros/bos_build/docs/release-ci.md)
