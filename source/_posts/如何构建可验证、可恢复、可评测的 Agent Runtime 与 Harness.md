---
title: 如何构建可验证、可恢复、可评测的 Agent Runtime 与 Harness
date: 2026-08-02 22:56:11
description: Agent 真正进入生产后，难点不再是能否调用模型和工具，而是结果能否验证、失败能否恢复、能力能否持续评测。本文从前端转 Agent/AI Infra 的视角，拆解 Runtime 与 Harness 的边界、核心对象、恢复等级、证据链和评测闭环，并给出可分阶段实施的工程方案。
categories:
  - [AI]
tags:
  - Agent
  - AI Infra
  - Agent Runtime
  - Harness
  - Evaluation
  - Checkpoint
cover: /images/agent-runtime-harness-architecture.webp
---

![可验证、可恢复、可评测的 Agent Runtime 与 Harness 参考架构](/images/agent-runtime-harness-architecture.webp)

*Agent Runtime / Harness 参考架构：以稳定的 Run Contract 包住不稳定的 Model 与 Provider，把验证、恢复和评测变成系统能力。*

最近我一直在想，所谓 Agent 工程到底比 Workflow 多了什么。

如果最终只是把 Prompt、LLM 和 Tool 连成一张图，再用 React Flow 画出来，它与 Dify、Coze 里的 Workflow 确实没有本质区别。真正让 Agent 从 Demo 走向生产的，不是节点更多，也不是图更炫，而是系统开始回答三类很不“智能”、却决定能否交付的问题：

- Agent 说任务完成了，我怎样证明它真的完成了？
- 执行到一半进程崩了，我怎样继续，而不是从头再赌一次？
- 换了模型、Prompt 或 Tool 后，效果究竟变好了还是变差了？

从前端转向 Agent / AI Infra 后，我越来越确信：值得投入的工程对象不是一个新的 Graph Editor，而是一套**可验证、可恢复、可评测的 Agent Runtime / Harness**。

## 一句话总结

**Runtime 负责让 Agent 跑起来，Harness 负责让这次运行可控制、可追踪、可恢复、可验收、可比较。**

服务器、容器、Sandbox 只是 Execution Environment；Agent Runtime / Harness 是运行在这些环境之上的控制系统。它不需要替模型思考，但必须约束模型如何访问状态、调用工具、产生副作用、等待审批、保存进度和交付结果。

我会用下面这条公式判断它是否真正落地：

```text
Production Agent
= Model / Agent SDK
+ Stable Run Contract
+ Controlled Side Effects
+ Durable State
+ Evidence-based Verification
+ Continuous Evaluation
```

少了后五项，通常只是一个可以演示的 Agent Loop。

## Runtime、Harness 和服务器运行环境不是一回事

“Runtime”这个词确实容易模糊，因为它在不同语境中指向不同层次。

| 层次 | 典型对象 | 负责什么 |
| --- | --- | --- |
| Execution Environment | Server、Container、Sandbox、Browser、Local Host | CPU、内存、网络、文件系统、进程隔离 |
| Model Runtime | Model API、Inference Server | 接收输入并生成 Token、Tool Call 或结构化输出 |
| Agent Runtime | Agent Loop、Run State、Tool Dispatch、Streaming、Handoff | 驱动一次 Agent 执行 |
| Agent Harness | Policy、Approval、Checkpoint、Trace、Evidence、Eval | 管理 Agent 执行的生产边界 |

现实中，Runtime 和 Harness 经常被混用。我的建议不是争论名词，而是把职责写清楚：

- 如果一个组件只负责 `while` 循环、调用模型和分发 Tool，它更接近 Agent Runtime。
- 如果它还拥有 Run ID、权限、审批、恢复、证据和评测，它已经是 Harness。
- 如果它只提供 Linux、Node.js、Python 和网络，那只是运行环境，不是 Agent Runtime。

这也是为什么部署了一个 Agent SDK，并不等于拥有了生产级 Runtime。SDK 解决的是“怎样调用”，Harness 解决的是“这次调用在业务上是否受控、可信、可持续演进”。

## 它与 Workflow 的真正区别

Workflow 描述**希望执行的路径**，Runtime / Harness 管理**实际发生的执行**。

```text
Workflow:  A → B → C

Runtime:   谁启动了 A？
           B 重试后会不会重复扣款？
           C 等待审批时状态存在哪里？
           进程重启后从哪一步继续？
           最终结果由什么证据证明？
           这次执行比上一个版本好在哪里？
```

因此，一张静态 Graph 可以是 Runtime 的输入，却不是 Runtime 本身。很多 Agent 任务甚至无法预先画出完整路径：模型会根据中间结果决定继续搜索、修改计划、调用其他工具或请求人工介入。

我会把两者关系理解成：

```text
Workflow / Graph = Execution Plan
Runtime / Harness = Execution Semantics + Control + Evidence
```

React Flow 只解决编辑和展示。如果产品真的需要让用户配置确定性流程，可以引入；如果当前目标是让 Agent 可靠执行，先做 React Flow 往往会把注意力带到节点拖拽、连线和表单配置上，最难的恢复与验收仍然没有答案。

## 先固定十个核心对象

一个可演进的 Harness，第一步不是选框架，而是定义自己的领域对象。至少要有下面十个：

| 对象 | 最小职责 | 不应该被什么替代 |
| --- | --- | --- |
| Workspace | 本次执行可见、可修改的工作边界 | 当前进程目录 |
| Revision | 执行绑定的输入与代码版本 | “最新版本” |
| Run | 一次可定位的端到端执行 | Provider 的 Session ID |
| Step | 可重试、可检查的状态转移 | 一条自然语言日志 |
| Tool Call | 结构化的工具调用请求 | 模型输出的一段文本 |
| Permission / Approval | 谁允许什么副作用发生 | Prompt 里的“请小心” |
| Checkpoint | 可恢复的 Run State 快照 | 聊天记录 |
| Event / Trace | 按时间记录发生了什么 | 最终回答 |
| Artifact / Evidence | 支撑完成声明的产物与证据 | Agent 自述“已完成” |
| Eval Result | 对结果、过程、成本与韧性的评分 | 主观体验“看起来不错” |

这里最容易混淆的是 Memory、State、Trace、Evidence 和 Eval：

- **Memory**：跨 Run 保留的偏好、事实和经验。
- **State**：当前 Run 此刻执行到哪里，以及继续所需的数据。
- **Trace**：这次 Run 按时间发生过哪些事件。
- **Evidence**：从产物和权威系统中选出的、能支撑某个 Claim 的证据。
- **Eval**：根据预先定义的标准，对结果或过程做判断。

聊天记录可以参与恢复，但它不是完整 State；Trace 可以帮助调查，但它不自动构成 Evidence；Evidence 能证明一次交付，却不能替代跨版本的 Eval。

## 三条必须守住的工程边界

### 1. Provider Adapter：隔离不稳定的 Agent 实现

模型、Agent SDK 和 Coding Agent 都会变化。业务系统不应该直接依赖某个 Provider 的 Thread ID、事件名和审批格式，而应该维护自己的稳定协议。

```ts
interface AgentRuntimeAdapter {
  start(input: StartRunInput): Promise<ExternalRunRef>;
  resume(input: ResumeRunInput): Promise<void>;
  cancel(runId: string): Promise<void>;
  stream(runId: string): AsyncIterable<AgentEvent>;
}

type AgentEvent =
  | { type: 'run.started'; runId: string }
  | { type: 'model.completed'; usage: Usage }
  | { type: 'tool.requested'; call: ToolCall }
  | { type: 'approval.required'; approval: ApprovalRequest }
  | { type: 'checkpoint.saved'; checkpointId: string }
  | { type: 'artifact.created'; artifact: ArtifactRef }
  | { type: 'run.completed'; claims: Claim[] }
  | { type: 'run.failed'; error: NormalizedError };
```

内部 `runId` 是事实主键，Provider Session ID 只是 `ExternalRunRef`。这样才能在不重写上层业务的情况下更换模型、SDK，甚至把一次任务路由到不同 Runtime。

适配层也不应该强行抹平所有差异。比如某个 Provider 支持原生 Resume，另一个只能重建 Context，应该通过 Capability 声明暴露：

```ts
type RuntimeCapabilities = {
  resumable: boolean;
  approval: 'native' | 'emulated' | 'none';
  streaming: boolean;
  sandbox: 'hosted' | 'local' | 'external';
};
```

“统一接口”不是假装大家能力相同，而是让差异变得可查询、可路由、可评测。

### 2. Tool Side Effect：所有真实动作经过受控边界

Agent 最危险的部分不是生成了一句错话，而是用错误参数真的执行了退款、发版、删库或发消息。

因此 Tool 不应只是一个普通函数。每次调用至少要绑定：

- `runId` 和 `stepId`；
- 调用者与 Workspace；
- 输入 Schema 和输出 Schema；
- 风险级别与所需 Permission；
- `idempotencyKey`；
- 超时、重试和补偿策略；
- 调用前后的权威状态快照；
- 产生的 Artifact 与 Evidence。

```ts
const result = await toolGateway.execute({
  runId,
  stepId,
  tool: 'release.create',
  input,
  idempotencyKey: `${runId}:${stepId}:release.create`,
  policy: { approval: 'required', timeoutMs: 60_000 },
});
```

重试只能作用于已声明幂等的边界。否则“失败后自动重试”可能变成重复扣款、重复发信或重复发布。

### 3. Run State：状态属于系统，不属于某次进程

如果 Node.js 进程一退出，任务就永远丢失，那么它没有真正的恢复能力。

Run State 至少应包含：

```ts
type RunState = {
  runId: string;
  status: 'queued' | 'running' | 'waiting_approval' |
          'suspended' | 'completed' | 'failed' | 'cancelled';
  workspace: WorkspaceRef;
  revision: RevisionRef;
  runtime: RuntimeRef;
  currentStep?: string;
  pendingToolCalls: ToolCall[];
  approvals: ApprovalState[];
  artifacts: ArtifactRef[];
  providerState?: unknown;
  contractVersion: string;
  updatedAt: string;
};
```

`contractVersion` 很重要。OpenAI Agents SDK 的 Human-in-the-loop 文档已经支持序列化 `RunState` 后跨请求恢复，同时也提醒：长期挂起任务需要把 Agent 定义和 SDK 版本与序列化状态一起管理。换句话说，Checkpoint 不只是保存 JSON，还要知道它由哪一版代码解释。

## 可验证：不要相信 Agent 的完成声明

“任务已完成”只是一条 Claim，不是事实。

真正的验证应该回到权威系统重新读取状态。例如 Coding Agent 说“修复完成”，Harness 应检查：

```text
Claim: 修复已完成
  ├─ Evidence: git diff 中只有目标范围的修改
  ├─ Evidence: 目标测试从 fail 变为 pass
  ├─ Evidence: 原有回归测试仍然通过
  ├─ Evidence: 构建产物成功生成
  └─ Acceptance: 目标环境 Smoke Test 通过
```

如果是退款 Agent，Evidence 应来自支付系统的退款状态与交易号；如果是发布 Agent，Evidence 应来自构建产物、Deployment ID 和线上健康检查，而不是 Tool 返回的“请求已提交”。

我建议把完成逻辑设计成显式的 Claim Verifier：

```ts
type Claim = {
  type: 'code.fixed' | 'release.online' | 'refund.completed';
  subject: string;
  evidence: EvidenceRef[];
};

interface ClaimVerifier {
  verify(claim: Claim, context: VerificationContext): Promise<{
    passed: boolean;
    checks: CheckResult[];
  }>;
}
```

这使“完成”从自然语言变成可执行的 Acceptance Contract。

Trace 与 Evidence 也要分开。OpenAI Agents SDK 的 Tracing 会记录 LLM generation、Tool Call、Handoff、Guardrail 和自定义事件，这非常适合调试和监控；但完整 Trace 只是原始执行记录。Evidence 是从 Trace、Artifact 和权威状态中挑出的、足以支撑 Claim 的最小集合。

## 可恢复：Checkpoint 只是起点

恢复能力可以按五个等级理解：

| 等级 | 能力 | 主要问题 |
| --- | --- | --- |
| L0 | 整个 Run 从头重试 | 重复成本高，副作用不安全 |
| L1 | 恢复 Provider Session | 能接着聊，不一定能恢复业务状态 |
| L2 | 按 Step 保存 Checkpoint | 能从明确边界继续 |
| L3 | Checkpoint + 幂等 Tool + Workspace Snapshot | 可以安全应对进程崩溃和重复消息 |
| L4 | Checkpoint 可跨 Runtime / Provider 迁移 | 通用性最高，成本和约束也最高 |

大多数生产系统应该先做到 L3，不必一开始追求 L4。

LangGraph 的 Persistence 明确区分 Checkpointer 与 Store：前者保存单个 Thread 的 Graph State，用于中断恢复、Human-in-the-loop、Time Travel 和 Fault Tolerance；后者保存跨 Thread 的长期数据。官方文档也直接提醒，`MemorySaver` 在进程重启后会丢失，生产环境要使用持久化 Checkpointer。

这再次说明：Conversation Memory 不是 Recovery State。

一个安全的恢复循环应该是：

```text
读取 Checkpoint
  ↓
校验 Contract Version 与 Workspace Revision
  ↓
检查上一个 Side Effect 是否已经生效
  ↓
用 Idempotency Key 决定复用结果还是重试
  ↓
恢复 Agent / Provider State
  ↓
继续下一步并写入新 Checkpoint
```

对于持续数小时、数天，且横跨多个服务的业务流程，可以把 Agent Framework 与 Temporal 组合使用：Agent Framework 负责模型循环和 Tool 选择，Temporal 负责 Durable Execution、Timer、Retry 与跨服务状态。Temporal 的官方定义强调，即使发生进程崩溃、网络故障或基础设施中断，应用也能从离开的地方恢复。它不是 Agent Framework 的替代品，而是更底层的可靠执行基础设施。

## 可评测：不要只评最终回答

Agent 的输出可能正确，过程却越权、昂贵或不可恢复；也可能过程合理，但外部系统最终没有生效。因此评测至少要分四层：

| 层次 | 关键问题 | 指标示例 |
| --- | --- | --- |
| Outcome | 任务最终是否真的成功 | Task Success、Test Pass、State Verification、人工验收率 |
| Process | Agent 走的路径是否合理 | Tool Selection、参数正确率、无效循环、越权次数、人工接管率 |
| Cost | 成功花了多少资源 | Latency、Token、Tool Call、Compute、Cost per Success |
| Robustness | 遇到故障还能否完成 | Resume Success、重复消息、超时、部分失败、Checkpoint 恢复率 |

OpenAI 的评测最佳实践强调 Eval-driven Development、任务特定数据集、完整日志、自动评分与持续评测，并明确反对“感觉它能工作”的 Vibe-based Evals。LangSmith 也把 Agent Eval 拆成 Final Response、Single Step 和 Trajectory；Trajectory 可以检查工具调用的顺序、集合与完整执行路径。

我倾向于组合三种 Grader：

1. **Code-based Grader**：测试、Schema、状态查询、静态规则，便宜、确定、可重复。
2. **Model-based Grader**：Rubric、Pairwise、复杂语义质量，覆盖开放式任务。
3. **Human Grader**：领域专家抽检和校准，防止自动评分体系自我欺骗。

Anthropic 的 Agent Eval 实践也采用这三类，并指出模型评分需要由人工校准。我的优先级是：能用权威状态和代码判定的，绝不先交给 LLM Judge；只有难以结构化的质量维度，才使用模型评分。

### 评测数据从哪里来

最初不需要几千条 Benchmark。可以先建立 20–30 个真实任务，覆盖：

- 高频正常路径；
- 历史失败案例；
- 权限与审批边界；
- Tool 超时和部分失败；
- 进程重启后的恢复；
- 重复消息与幂等；
- 模糊需求和人工接管；
- 成本或延迟极端值。

每个 Case 固定 Workspace Revision、输入、可用 Tool、成功条件和预算，再对 Provider、Model、Prompt、Tool Version 做矩阵比较。非确定性的 Agent 至少重复运行 3–5 次，不要拿一次成功当能力结论。

```text
Eval Case
  ├─ Fixture: Workspace + Revision + External State
  ├─ Input: User Intent
  ├─ Policy: Allowed Tools + Approval Rules
  ├─ Expected Outcome: Acceptance Contract
  ├─ Process Constraints: Forbidden / Required Behaviors
  └─ Budget: Time + Token + Cost
```

Run、Trace、Evidence 和 Eval Result 必须共用同一个 `runId`。否则评测只剩一张分数表，无法回到具体轨迹解释为什么变好或变差。

## 最小可落地架构

我不建议一开始自研完整平台。先做一个薄 Harness，把不可替代的控制权握在自己手里：

```text
User / Trigger
      ↓
Harness API
      ├─ Run Store / Event Log
      ├─ Policy & Approval
      ├─ Workspace & Revision
      ├─ Tool Gateway
      ├─ Checkpoint Store
      ├─ Artifact & Evidence
      └─ Eval Runner
              ↓
      Runtime Adapter
        ├─ OpenAI Agents SDK
        ├─ LangGraph
        └─ Other Agent / Coding Runtime
              ↓
      Model + Tools + Execution Environment
```

最小技术选型可以非常朴素：

| 能力 | 第一阶段建议 |
| --- | --- |
| Run / Event / Checkpoint | PostgreSQL；本地单机可先 SQLite |
| Artifact | Object Storage 或受版本控制的 Workspace |
| Trace | OpenTelemetry，或先接 Provider 自带 Tracing 再导出 |
| Tool Contract | JSON Schema + 统一 Gateway + Idempotency Key |
| Approval | 数据库状态机 + 通知，不要让请求连接一直挂着 |
| Eval | 测试框架 + 自建 Case Schema；需要可视化再接评测平台 |
| Durable Workflow | 长周期、跨服务场景再引入 Temporal |
| Graph UI | 只有用户确实要编辑流程时再引入 React Flow |

### 现成框架怎么选

**OpenAI Agents SDK** 适合快速建立 Agent Loop，并直接获得 Tool、Handoff、Guardrail、Tracing 和 Human-in-the-loop。它的序列化 RunState 可以承担一部分恢复能力，但业务 Run、Tool 幂等、权威状态验证和跨 Provider 协议仍应由自己的 Harness 管理。

**LangGraph** 适合状态和分支需要显式建模的 Agent。Graph State、Checkpoint、Interrupt、Time Travel 和持久化是它的强项。代价是图与状态 Schema 会进入核心代码，不要为了“看起来工程化”把开放式 Agent 强行拆成几十个节点。

**Temporal** 适合长周期、高价值、跨服务、必须可靠恢复的业务执行。它解决 Durable Execution，不负责 Prompt、Tool Selection 和 Agent Quality，通常与 Agent SDK 组合，而不是二选一。

**LangSmith、OpenAI Tracing / Evals 等平台**可以承担观测和评测的一部分，但不要让平台生成的 Trace ID 成为业务主键。保留自己的 Run Contract，才能迁移、对比和审计。

所以我的默认组合不是“选一个大而全框架”，而是：

```text
自有 Thin Harness
+ 一个 Agent Runtime / SDK
+ 持久化数据库
+ 统一 Tool Gateway
+ OpenTelemetry / Provider Trace
+ 任务级 Eval Suite
```

## 分五个阶段落地

### Phase 1：先让每次运行可定位

建立 `runId`、Run 状态机、统一 Event Schema 和基本 Trace。所有 Model Call、Tool Call、Artifact、Approval 都关联同一个 Run。

验收标准：任意一次失败都能回答“谁在什么时候，用什么版本，调用了什么，在哪一步失败”。

### Phase 2：控制所有副作用

把外部动作收口到 Tool Gateway，加入 Schema、Permission、Approval、Timeout、Idempotency 和调用前后状态。

验收标准：重复投递同一 Tool Call 不会造成重复业务结果；高风险操作不能绕过审批。

### Phase 3：做到进程级恢复

持久化 Run State 和 Checkpoint，绑定 Workspace Revision、Agent Definition Version 与 Runtime Version，补 Resume、Cancel 和过期策略。

验收标准：在关键 Step 后强制杀掉进程，重启后可以继续，且不重复已完成的副作用。

### Phase 4：建立 Evidence-based Acceptance

为核心 Claim 建立 Verifier，从权威系统重新读取结果，生成最小 Evidence Bundle 与 Acceptance Result。

验收标准：Agent 自述与真实状态不一致时，Run 不能被标记为业务完成。

### Phase 5：用 Eval 驱动演进

沉淀真实 Case，记录 Outcome、Process、Cost、Robustness，比较不同 Model、Prompt、Tool 和 Runtime，逐步加入线上采样与回归门禁。

验收标准：每次关键变更都能回答“成功率、成本、延迟和失败类型发生了什么变化”。

这五个阶段不需要 Graph Editor。等到系统已经拥有可靠的执行语义，再把适合人工配置的确定性部分可视化，React Flow 才会成为产品能力，而不是工程完成度的幻觉。

## 常见反模式

### 把 Provider Session ID 当自己的 Run ID

一旦切换 Provider 或同一业务 Run 包含多段模型执行，关联关系就会断裂。Provider ID 应只是外部引用。

### 把聊天历史当 Checkpoint

聊天历史没有完整表达审批状态、已执行副作用、Workspace Revision 和待处理 Tool Call，恢复时容易重复执行或错用环境。

### 把 Trace 当 Evidence

Trace 证明“系统记录了这个动作”，不证明“业务结果真的生效”。Evidence 必须能回到 Artifact 或权威状态。

### 只评最终回答

最终答案正确，不代表过程中没有越权、泄露、无效循环或不可接受的成本。Agent Eval 必须覆盖 Trajectory 和外部结果。

### 一开始追求通用编排平台

没有真实 Case 时，通用节点、DSL 和图编辑器只会提前固化错误抽象。先用 20–30 个真实任务逼出稳定契约，再抽象框架。

## 要不要用 / 我的判断框架

### 值得建设

- Agent 会修改代码、发布、退款、发消息或操作业务系统；
- 单次任务长、成本高，中断后不能从头开始；
- 需要人工审批、权限隔离、审计或合规；
- 需要比较多个 Model、Provider 或 Agent Runtime；
- 团队已经遇到“Demo 能跑，线上偶发失败却解释不清”的问题。

### 可以再等

- 只是单轮问答或内部 POC；
- 没有外部副作用，失败后整次重试成本很低；
- 需求和 Tool 还在快速变化，连成功标准都没有定义；
- 当前瓶颈是基础能力，而不是稳定性、治理或评测。

### 重点关注

如果现在只能做一件事，我不会先引入 React Flow，也不会先造一个通用 Graph Engine。我会先建立统一 `runId` 和 Event Contract，让 Tool Call、Checkpoint、Artifact、Evidence 与 Eval Result 能围绕同一次运行串起来。

这条链一旦成立，后面接 OpenAI Agents SDK、LangGraph、Temporal 或其他 Runtime 都只是适配问题；这条链没有成立，框架越多，只会制造更多无法解释的状态。

我对 Agent Runtime / Harness 的最终判断是：它不是一个新潮名词，也不该被包装成“更高级的 Workflow”。它真正有价值的地方，是把概率性的 Agent 包进确定性的工程边界里——**让每次执行有身份，让每个副作用受控制，让每次中断能继续，让每个完成声明有证据，让每次升级都能被测量。**

这才是 Agent 工程开始进入 AI Infra 的分界线。

## 参考资料

- [OpenAI Agents SDK：Tracing](https://openai.github.io/openai-agents-js/guides/tracing/)
- [OpenAI Agents SDK：Human-in-the-loop](https://openai.github.io/openai-agents-js/guides/human-in-the-loop/)
- [OpenAI：Evaluation best practices](https://developers.openai.com/api/docs/guides/evaluation-best-practices)
- [LangGraph：Persistence](https://docs.langchain.com/oss/javascript/langgraph/persistence)
- [LangSmith：Agent trajectory evaluations](https://docs.langchain.com/langsmith/trajectory-evals)
- [Temporal：Durable Execution](https://docs.temporal.io/)
- [Anthropic：Demystifying evals for AI agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)
