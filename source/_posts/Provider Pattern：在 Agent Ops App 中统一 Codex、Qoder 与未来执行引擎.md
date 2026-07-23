---
title: Provider Pattern：在 Agent Ops App 中统一 Codex、Qoder 与未来执行引擎
date: 2026-07-21 14:30:00
description: 当 Agent Ops App 同时接入 Codex、Qoder，未来还要支持 Hermes 时，真正的难点不是多写几个 SDK 调用，而是阻止 Provider 私有协议渗透业务层。本文结合桌面 Agent Host 实践，拆解 Provider、Adapter、Registry、Capability、Session Pin 与 Runtime Resolver 的职责，并给出一套可持续扩展的实现框架。
categories:
  - [AI]
tags:
  - Provider Pattern
  - Adapter Pattern
  - Agent Ops
  - AI Infra
  - Codex
  - Qoder
  - Architecture
cover: /images/provider-pattern-agent-ops.webp
---

最近在做一个 Agent Ops 桌面应用时，我先接了 Codex，下一步要接 Qoder，后面还可能接 Hermes。最开始这看起来只是一个 SDK 集成问题：再写一个 `QoderClient`，在 UI 里加个选择框就结束了。

但我很快发现，真正的问题不是“怎么调用第二个 Agent”，而是：**当第三个、第四个执行引擎出现时，业务代码是否还需要跟着修改？**

如果 UI 认识 `thread/start`，Workflow 判断 `provider === 'qoder'`，Session 直接拿 Provider Session ID 当主键，那么所谓“多 Provider”只是把多个私有协议揉进了同一个应用。接入得越多，系统越难维护。

我最后采用的是 Provider Pattern：把 Codex、Qoder、Hermes 看作可替换的执行引擎，用 Adapter 吸收协议差异，再通过 Registry、Capability 和 Session Pin 建立稳定的 Agent Ops 控制面。

## 一句话总结

**Provider 是能力提供者，Adapter 是把能力接入统一契约的翻译层。Agent Ops App 应该拥有 Session、权限、Workflow 和审计，Provider 只拥有原生执行上下文；新增 Provider 时，理想状态是只增加 Adapter 和 Projection，而不是修改整条业务链路。**

## Provider 不是 Adapter

这两个词经常同时出现，也最容易混在一起。

| 概念 | 回答的问题 | 在 Agent Ops App 中的例子 |
|---|---|---|
| Provider | 谁提供执行能力 | Codex、Qoder、Hermes |
| Provider Adapter | 怎样接入这个 Provider | `CodexAdapter`、`QoderAdapter` |
| Provider Contract | App 要求所有 Provider 遵守什么接口 | `createSession`、`runTurn`、`interrupt` |
| Provider Registry | 运行时选择哪个 Adapter | `providerId → ProviderAdapter` |

Provider 更接近一种架构角色，不是 GoF 设计模式。支付系统里的支付宝、微信支付是 Payment Provider；对象存储里的 S3、OSS、GCS 是 Storage Provider；Agent Ops 中的 Codex、Qoder、Hermes就是 Agent Provider。

Adapter 则是经典的 Adapter Pattern。它不负责推理，只负责把 Provider 的原生接口翻译成 App 的统一语义。

```text
Agent Ops App
    ↓ Provider Contract
Provider Adapter
    ↓ Provider Native API
Codex / Qoder / Hermes
```

一句话区分：**Provider 是被接入的对象，Adapter 是负责接入它的代码。**

## 为什么不能在 UI 里直接判断 Provider

Codex 和 Qoder 都支持会话、流式输出和中断，但原生语义并不相同。

Codex App Server 可能使用：

```text
thread/start
turn/start
turn/interrupt
item/agentMessage/delta
```

Qoder Agent SDK 则可能使用：

```text
query()
options.sessionId
options.resume
query.interrupt()
stream_event
```

最直接的实现是在业务层写分支：

```ts
if (provider === 'codex') {
  await codex.startTurn(...);
} else if (provider === 'qoder') {
  await qoder.query(...);
}
```

这个写法在第二个 Provider 时成本很低，但它会迅速扩散到 UI、Session、Workflow、Approval、错误处理和测试。最终每个功能都要知道所有 Provider 的私有细节。

更稳妥的做法是先定义 Provider-neutral Contract：

```ts
type ProviderId = string;

interface ProviderAdapter {
  readonly id: ProviderId;

  probe(): Promise<ProviderDescriptor>;
  createSession(input: CreateSessionInput): Promise<string>;
  runTurn(
    input: RunTurnInput,
    emit: (event: AgentEvent) => void,
  ): Promise<void>;
  interrupt(providerSessionId: string): Promise<void>;
  close(providerSessionId: string): Promise<void>;
}
```

业务层只处理 `AgentEvent`：

```ts
type AgentEvent =
  | { type: 'turn.started'; turnId: string }
  | { type: 'message.delta'; text: string }
  | { type: 'tool.started'; tool: string }
  | { type: 'tool.completed'; tool: string; status: string }
  | { type: 'turn.completed' }
  | { type: 'session.failed'; message: string };
```

Codex 的 `item/agentMessage/delta` 和 Qoder 的 `stream_event` 都在 Adapter 内转换成 `message.delta`。上层不需要知道事件来自哪个 SDK。

## Provider Pattern 实际组合了哪些模式

我把它叫作 Provider Pattern，但严格来说，它是一组模式的组合。

| 模式 | 解决的问题 |
|---|---|
| Adapter Pattern | 把不同 Provider 的接口和事件翻译成统一语义 |
| Strategy Pattern | 在运行时选择 Codex、Qoder 或 Hermes 执行任务 |
| Registry Pattern | 管理 `providerId` 与 Adapter 实例的映射 |
| Ports and Adapters | 让 Agent Ops 领域层不依赖外部 SDK |
| Anti-Corruption Layer | 阻止 Provider 私有概念污染业务模型 |

Registry 的实现可以很小：

```ts
class ProviderRegistry {
  #adapters = new Map<string, ProviderAdapter>();

  constructor(adapters: ProviderAdapter[]) {
    for (const adapter of adapters) {
      if (this.#adapters.has(adapter.id)) {
        throw new Error(`Duplicate provider: ${adapter.id}`);
      }
      this.#adapters.set(adapter.id, adapter);
    }
  }

  get(providerId: string): ProviderAdapter {
    const adapter = this.#adapters.get(providerId);
    if (!adapter) throw new Error(`Unsupported provider: ${providerId}`);
    return adapter;
  }
}
```

这里我刻意把 `ProviderId` 定义成 `string`，而不是：

```ts
type ProviderId = 'codex' | 'qoder';
```

封闭 union 在小型应用里类型体验很好，但对可扩展 Provider 系统意味着每接入一个 Provider 都要修改公共契约。真正的约束应该来自 Registry、配置校验和兼容矩阵，而不是把已知 Provider 列表写死在领域类型中。

## Session 必须由 Agent Ops App 持有

这是多 Provider 架构里最重要的所有权边界。

Codex 有 Thread ID，Qoder 有 Session ID，Hermes 未来也会有自己的会话标识。但这些 ID 都不能直接成为 Agent Ops 的业务主键。

```text
应用 Session ID
    ├── providerId: qoder
    ├── providerVersion: 1.0.47
    ├── adapterVersion: 0.1.0
    ├── workspaceRevisionId
    ├── permissionProfileId
    └── providerSessionId: <Qoder native ID>
```

原因很直接：

- Provider Session ID 的格式和生命周期不受 App 控制。
- Provider 可能删除、迁移或无法恢复原生 Session。
- Workflow、审批和审计需要跨 Provider 保持统一身份。
- 用户切换 Provider 时，业务记录不能跟着换主键。

因此 Agent Ops Session 是权威记录，Provider Session 只是外部引用。

创建 Session 时还应该固定 Provider 版本、Workspace Revision、权限配置和资产投影版本。否则同一个 Session 运行到一半，系统 CLI 自动升级，前后两次 Turn 可能已经不再遵守相同协议。

## Desktop、CLI、SDK 和 Adapter 要分层

Codex 和 Qoder 都可能同时提供 Desktop、CLI 和 SDK，这几个东西不能混为一谈。

| 层级 | 作用 | 是否进入 Agent Runtime 主链路 |
|---|---|---|
| Desktop | 用户直接操作的产品界面 | 否，适合 Open/Handoff |
| CLI | 可执行 Runtime、登录和诊断入口 | 是 |
| SDK | 管理 CLI、协议和结构化事件 | 优先使用 |
| Adapter | Agent Ops 内部的转换边界 | 是 |

桌面端不适合成为 Agent Ops 的执行依赖：它的生命周期由用户控制，可能自动升级，窗口里的会话也不等于 Agent Ops Session。

真正的调用链应该是：

```text
Agent Ops App
  → Provider Adapter
  → SDK / CLI structured protocol
  → Provider Runtime
```

Desktop 只负责用户交接：

```text
Open in Codex
Open in Qoder
Handoff current task
```

## Runtime Resolver：版本稳定性的关键一层

只写 `spawn('codex')` 或 `spawn('qodercli')` 很方便，但这会默认依赖用户 `PATH` 中的版本。系统 CLI 一旦自动升级，Adapter 可能在用户毫无感知的情况下失效。

我倾向于给每个 Provider 增加 Runtime Resolver：

```ts
type ResolvedProviderRuntime = {
  executablePath: string;
  source: 'override' | 'bundled' | 'system';
  version: string;
};
```

解析顺序是：

```text
显式环境变量覆盖
  → App/SDK bundled runtime
  → 系统 CLI 开发兜底
```

以 Qoder 为例，SDK 可能锁定一个经过验证的 bundled CLI，系统新版本只用于登录和人工诊断。Codex 也应该采用相同思路，只是在真正随 App 分发 binary 前，还需要确认授权、签名、公证和登录态复用边界。

版本不能只在日志里打印一次，而要进入 Session Pin 和 Compatibility Manifest：

```json
{
  "provider": "qoder",
  "providerRuntimeVersion": "1.0.47",
  "providerAdapterVersion": "0.1.0",
  "sidecarProtocolVersion": 1,
  "capabilities": ["stream", "interrupt", "resume"]
}
```

## Capability 不要伪装成统一能力

统一接口不代表所有 Provider 的能力完全相同。

有些 Provider 支持 resume，有些只支持重建上下文；有些能 fork Session，有些能提供结构化 Tool Approval；有些只有文本流，没有稳定的 Tool Event。

错误做法是用文本解析或业务分支伪造能力。正确做法是让 `probe()` 返回 Capability：

```ts
type ProviderDescriptor = {
  id: string;
  version: string;
  status: 'ready' | 'unavailable';
  capabilities: Array<
    'stream' | 'interrupt' | 'resume' | 'fork' | 'toolApproval'
  >;
};
```

上层根据数据决定是否展示“恢复”“Fork”或“审批”按钮，而不是判断 Provider 名称：

```ts
if (provider.capabilities.includes('resume')) {
  showResumeAction();
}
```

这能让未来 Hermes 接入时只声明真实能力，不必模仿 Codex 或 Qoder。

## 权限不能交给 Provider 自己决定

Provider Adapter 解决的是兼容性，不自动解决安全问题。

即使 Codex、Qoder 都有自己的 Permission Mode，Agent Ops App 仍然必须持有上层权限策略。Provider 原生审批只能进一步收紧，不能放宽组织策略已经拒绝的动作。

我当前采用的 MVP 边界是：

1. 原生 Core 验证 Workspace Grant。
2. Core 生成有界的只读 Workspace Context。
3. Adapter 禁用 Provider 原生文件、Shell、网络和 MCP 副作用工具。
4. Provider 只负责推理与流式输出。
5. 未来开放 Tool 时，必须先转换成 Canonical Tool Request，再由 Core 执行。

这能避免一个常见误区：UI 看起来使用了 read-only，但 Provider Runtime 实际继承了完整用户目录和 Shell 权限。

## 新增 Hermes 时应该改多少代码

一个健康的 Provider 架构应该用“新增代码”扩展，而不是到处修改旧代码。

理想的 Hermes 接入包括：

```text
hermes-adapter.ts
hermes-runtime-resolver.ts
hermes-materializer.ts
hermes-contract-fixtures/
```

然后注册：

```ts
new ProviderRegistry([
  new CodexAdapter(),
  new QoderAdapter(),
  new HermesAdapter(),
]);
```

不应该修改：

- Agent Chat 的事件渲染逻辑
- Workflow Run 状态机
- Approval 数据模型
- Audit Event 格式
- Asset 的事实源模型

如果新增 Hermes 还需要修改以上模块，说明 Provider 私有语义已经穿透 Adapter 边界。

## 最容易踩的五个坑

### 1. 把 Provider 和 Model 混为一谈

Provider 是执行系统，Model 只是其中一个配置。Qoder 可能动态选择多个 Model，Codex 也可能切换推理配置。Session 应固定 Provider，Model 是否固定则由产品策略决定。

### 2. 一个 Adapter 只保存一个全局 Session

类似 `#threadId` 的单例字段只能支持一个会话。生产实现应按 Provider Session ID 管理进程、事件监听器、Pending Request 和取消状态。

### 3. UI 出现 Provider 私有事件

一旦 UI 直接处理 `item/agentMessage/delta`，Adapter 就已经失去意义。Provider 原生类型应该止步于 Adapter 文件。

### 4. 依赖系统 CLI 自动升级

开发机上很方便，生产环境中却是协议漂移来源。至少要做 Runtime Resolver、版本 Probe、兼容范围校验和 Session Pin。

### 5. 为了“统一”而虚构能力

不支持 resume 就明确返回不支持，不要偷偷重放 Prompt 并告诉上层“恢复成功”。能力差异应该由数据表达，不能靠语义造假。

## 要不要用 Provider Pattern：我的判断框架

如果应用只调用一个稳定 API，且没有切换执行引擎的计划，直接封装一个 Client 往往足够，不必为了模式而模式。

但满足下面任意三项时，我会尽早建立 Provider 边界：

- 已经有两个以上执行引擎。
- Provider 拥有不同的 Session 和事件协议。
- SDK/CLI 更新频繁，需要独立兼容测试。
- UI、Workflow、审批和审计必须保持统一。
- 需要运行时选择 Provider。
- 未来可能接入企业内部或自托管 Agent Runtime。

对 Agent Ops App 来说，我的结论是“值得上”。因为它的核心价值不是把某个 Agent 包一层 UI，而是拥有跨 Provider 的控制面：资产、权限、Session、Workflow、Evidence 和审计。

Provider 可以替换，Adapter 可以升级，CLI 可以演进；但 Agent Ops App 的领域模型和治理边界必须保持稳定。做到这一点以后，Codex、Qoder、Hermes 才真正只是可替换的执行引擎，而不是反过来绑架整个产品架构。
