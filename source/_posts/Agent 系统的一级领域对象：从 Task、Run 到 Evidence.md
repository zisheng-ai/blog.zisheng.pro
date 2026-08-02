---
title: Agent 系统的一级领域对象：从 Task、Run 到 Evidence
date: 2026-08-02 17:20:00
description: Agent、Task、Run、Tool、Memory、Policy、Evidence 经常被放在同一张架构图里，却不是同一层概念。本文交叉比对 OpenAI Agents SDK、Anthropic、A2A、MCP、LangGraph 与 W3C PROV，给出一级领域对象的判定标准、十二个核心对象及生产化建模顺序。
categories:
  - [AI Infra]
tags:
  - Agent
  - Domain Model
  - DDD
  - Evidence
  - A2A
  - MCP
  - AI Infra
cover: /images/agent-domain-model.webp
---

![Agent 系统的一级领域对象：Agent、Task、Run、Artifact、Evidence 与 Evaluation 的关系](https://blog.zisheng.pro/images/agent-domain-model.webp)

*一套跨框架的 Agent 领域模型：Agent 承担责任，Task 描述工作，Run 记录执行，Artifact 承载结果，Evidence 和 Evaluation 决定结果是否可信。*

最近我反复遇到 `Evidence`。它出现在知识检索、Agent 执行、评测、审计和业务验收中，与 `Agent`、`Task`、`Run`、`Memory`、`Policy` 并列。问题随之而来：这些词到底是不是同一层概念？一个 Agent 系统真正需要维护哪些一级领域对象？

如果直接从某个 SDK 抄类名，答案会随着框架切换而变化。OpenAI Agents SDK 强调 Agent、Runner、Tools、Handoffs、Guardrails 和 Sessions；A2A 暴露 Agent Card、Task、Message、Artifact；MCP 提供 Tools、Resources、Prompts；LangGraph 围绕 State、Thread、Checkpoint 和 Memory 组织运行。它们都合理，但它们描述的是不同边界。

我更关心跨框架后仍然成立的业务语义。因为只有这些对象，才值得进入 Agent Platform 的数据库、API、权限、状态机和审计模型。

## 一句话总结

我会把 Agent 系统的核心领域模型收敛为十二个对象：

```text
Actor      Agent / Identity
Work       Goal / Task / Run
Execution  Capability / Context
Control    Policy
Outcome    Artifact / Claim
Trust      Evidence / Evaluation
```

它们组成一条完整链路：

```text
Identity 授权 Agent
Agent 为 Goal 承接 Task
Task 产生一次或多次 Run
Run 在 Policy 约束下使用 Capability 和 Context
Run 产出 Artifact，并提出 Claim
Evidence 支撑 Claim
Evaluation 根据验收标准作出判断
```

`Tool`、`Prompt`、`Model`、`Runtime`、`Message` 和 `Trace` 同样重要，但默认属于实现、协议或可观测性对象，不必全部提升为一级领域对象。

## “一级领域对象”不是现成标准

“一级领域对象”并不是 Agent 行业已经发布的正式术语。这里采用的是领域建模视角：当一个对象需要被系统长期、独立地管理，就把它当作 First-class Domain Object，而不只是某张配置表里的字段。

我用五个问题判断：

1. 它是否拥有稳定 ID，能被 API、权限和审计记录独立引用？
2. 它是否有独立生命周期，而不是伴随一次函数调用即消失？
3. 它是否承载业务规则或状态转换？
4. 它是否会被多个模块共同使用，而不属于某个框架的内部实现？
5. 更换 Model、Runtime 或 Agent Framework 后，它是否仍然存在？

例如 `Run` 会被暂停、恢复、取消、重试和验收，显然应当独立建模。某个模型的 `temperature` 不具备这些特征，它只是配置值。

这也意味着“一级”不是永恒不变的。`Workspace` 对普通客服 Agent 可能只是 Context 的一个字段，对 Coding Agent 却可能有独立权限、Revision、锁和生命周期，此时就应升级为一级对象。

## 先从跨框架的共同部分看

### W3C PROV 给出了最小骨架

Agent 领域不是从零开始发明。W3C 的 [PROV-O](https://www.w3.org/TR/prov-o/) 用三个起点描述可追溯系统：

- `Agent`：对某项活动或实体承担责任的主体；
- `Activity`：在一段时间内发生、使用或生成实体的活动；
- `Entity`：具有相对固定特征的物理、数字或概念对象。

映射到 Agent 系统里，最朴素的关系就是：

```text
Agent 负责 Run
Run 使用 Context / Resource
Run 生成 Artifact / Evidence
```

这个骨架解释了为什么 Evidence 不属于“知识库技术”或者“日志技术”本身。Evidence 是带有来源、生成活动和责任主体的 Entity，价值来自它与 Claim、Run、Agent 的关系。

### A2A 把 Task 和 Artifact 提升为协议对象

[A2A 1.0 规范](https://a2a-protocol.org/latest/specification/)把 `Task` 定义为有唯一 ID、状态和生命周期的基本工作单元；Agent 执行后产生的文档、图片或结构化数据进入 `Artifact`。`Message` 负责通信，`Context` 负责关联一组 Task 和 Message。

这个区分非常关键：

```text
Message：一次沟通
Task：需要跟踪的工作
Artifact：工作产生的交付物
```

聊天记录不能代替任务状态，最终回复也不能自动等于正式交付物。

### MCP 描述能力接口，不负责完整业务模型

[MCP 架构](https://modelcontextprotocol.io/docs/learn/architecture)定义了三个核心 Server Primitive：

- Tools：执行动作；
- Resources：向模型提供上下文数据；
- Prompts：提供可复用的交互模板。

它回答的是“Agent 如何接入能力与资源”，不负责定义 Goal、Task、Run、Evidence 或 Evaluation。A2A 规范也把两者定位为互补协议：MCP 面向工具和资源连接，A2A 面向 Agent 之间的任务协作。

所以 `MCP Server` 不应成为业务领域模型的中心。它是 Capability 和 Resource 的一种接入方式。

### Agent Framework 暴露的是 Runtime Primitive

[OpenAI Agents SDK](https://openai.github.io/openai-agents-python/)把 Agent 定义为带有 Instructions、Tools、Handoffs 和 Guardrails 的 LLM，并由 Runner 执行 Agent Loop。它还提供 Session、Tracing、Human-in-the-loop 和可恢复的 Run State。

[Anthropic 的 Agent 架构总结](https://www.anthropic.com/engineering/building-effective-agents)则把 Agent 与 Workflow 分开：Workflow 走预定义代码路径，Agent 动态决定过程和工具使用。其基础构件是带 Retrieval、Tools、Memory 的 Augmented LLM；执行过程中，Agent 需要持续从工具结果、代码执行等环境反馈中取得 Ground Truth。

这些材料共同证明 Agent、Run、Capability、Context、Memory、Policy 和 Evaluation 的必要性，但 SDK 里的每个类并不等于领域对象。`Runner` 可以被替换，Task 和 Evidence 不能因为换了 Runtime 就消失。

把几套体系放在同一张表里，边界会更清楚：

| 来源 | 它显式建模的对象 | 对领域模型的启发 |
| --- | --- | --- |
| W3C PROV | Agent、Activity、Entity、Attribution、Derivation | 责任、活动、产物和来源关系是追溯骨架 |
| OpenAI Agents SDK | Agent、Tools、Handoffs、Guardrails、Session、Run State、Trace | Agent Loop 之外还需要运行状态、控制与上下文 |
| Anthropic | Agent、Workflow、Tools、Retrieval、Memory、Ground Truth、Evaluator | 自主决策必须接受环境反馈与独立评价 |
| A2A | Agent Card、Task、Message、Context、Artifact | 通信、工作、上下文和交付物需要分开 |
| MCP | Tools、Resources、Prompts | 能力、资源和交互模板属于接入协议边界 |
| LangGraph | State、Thread、Checkpoint、Memory | 长运行需要状态快照、恢复与跨会话记忆 |

这张表不是投票结果。某个概念在六套体系里都出现，不代表它一定是一级对象；反过来，Claim 没有被多数 Agent SDK 显式建模，也不代表生产系统可以省略它。最终仍要回到 ID、生命周期、业务规则和跨模块关系。

## 十二个核心领域对象

### 1. Agent：承担工作责任的持续主体

`Agent` 不应只是一段 System Prompt，也不只是“模型 + 工具列表”。它代表一个可被分配工作、拥有能力边界和版本、能够产生运行记录与结果责任的持续主体。

一个可管理的 Agent 至少要回答：

- 当前有效版本是什么；
- 可以承接哪些工作；
- 以谁的身份行动；
- 允许使用哪些能力；
- 失败、暂停或升级时由谁负责。

模型只是 Agent 的实现组件。把模型名直接当 Agent ID，会让版本、权限、责任和历史运行全部粘在供应商配置上。

### 2. Identity：谁在行动，代表谁行动

`Identity` 解决认证与责任归属。Agent 可以有自己的服务身份，也可能代表某个用户、团队或组织发起操作。

需要区分：

```text
Agent Identity：哪个 Agent 在执行
Principal：它代表谁行动
Credential：本次连接使用什么凭证
```

Credential 应短期、可轮换，Identity 和 Principal 则需要进入审计关系。否则系统只能知道“某个 Token 调用了接口”，无法说明谁授权了哪个 Agent 做什么。

### 3. Goal：希望世界达到什么状态

`Goal` 是期望结果，不是执行步骤。例如“让新版页面可被用户访问”是 Goal，“修改配置并运行发布命令”只是候选计划。

Goal 值得独立存在，是因为同一个 Goal 可能被拆成多个 Task，也可能在执行过程中改变计划，但成功条件不应随 Agent 的临时推理漂移。

一个可用 Goal 至少包含目标状态、作用范围、成功指标、截止条件和责任人。

### 4. Task：可分配、可跟踪的工作单元

`Task` 把 Goal 变成可以交给 Agent 的工作。A2A 把它作为协议核心，是因为长任务必须有稳定 ID、状态、历史和结果。

Task 与 Goal 的区别是：

```text
Goal：为什么做、做到什么程度
Task：谁承接哪一块工作、当前处于什么状态
```

多 Agent 系统里的 Delegation 或 Handoff，本质上是在 Agent 与 Task 之间改变责任关系，而不只是转发一条 Message。

### 5. Run：Task 的一次执行尝试

`Run` 是最容易被漏掉、也最需要独立建模的对象。同一个 Task 可能第一次执行超时，第二次恢复成功，第三次为了验证重新运行。它们属于一个 Task，却不是同一次 Run。

Run 应记录：

- 使用的 Agent Version、Context Snapshot 和 Policy Version；
- 开始、暂停、恢复、取消和结束状态；
- Tool Call、Handoff、异常和人工决定；
- Usage、成本、产物与证据引用。

OpenAI Agents SDK 的可序列化 `RunState` 和 LangGraph 的 Checkpoint 都在解决“运行如何暂停并恢复”，但领域层不应绑定它们的存储格式。

### 6. Capability：Agent 能完成哪类动作

`Capability` 描述业务能力，例如“查询订单”“修改代码”“发布静态站点”。`Tool`、`Skill`、`Model`、`Runtime` 和 MCP Server 是能力的不同实现或依赖。

```text
Capability：能做什么
Tool：通过什么接口做
Skill：按什么方法做
Model：由什么推理能力参与
Runtime：在哪个执行引擎里做
```

把 Capability 与实现分开，才能替换工具和 Runtime，同时保留任务路由、权限、评测与历史成功率。

### 7. Context：本次运行获准看到的工作集

`Context` 不是“把所有资料塞进上下文窗口”，而是某次 Run 使用的信息边界。它可以引用 Conversation、Workspace、Resource、知识片段、环境状态和用户输入。

Context 最好以 Snapshot 或明确 Revision 固定下来。否则运行结束后无法回答：Agent 当时看到了哪个版本的文件、哪段知识、哪个系统状态？

[LangGraph 的 Memory 文档](https://docs.langchain.com/oss/python/concepts/memory)把短期记忆放在线程状态中，并通过 Checkpoint 持久化；跨线程的长期记忆则单独存储。这个划分说明 Memory 不是 Context 的同义词：Context 是本次运行实际使用的集合，Memory 是可能被召回到未来 Context 的持久状态。

### 8. Policy：什么情况下允许怎么做

`Policy` 是对 Agent、Task、Capability、Context 和 Run 的约束规则，例如：

- 哪些数据可以发送给哪个 Model；
- 哪些 Tool 只能在指定 Workspace 使用；
- 哪些动作必须人工审批；
- 超过什么预算停止运行；
- 哪些 Evidence 缺失时不得宣布完成。

Guardrail 是 Policy 的一种执行机制，不是 Policy 本身。Prompt 里的“请不要执行危险操作”也不能替代可版本化、可判定、可审计的 Policy。

### 9. Artifact：Agent 实际交付了什么

`Artifact` 是 Run 产生的持久结果，例如代码 Diff、文档、报表、图片、构建包、结构化数据或发布清单。A2A 明确把 Artifact 与 Message 分开，这个设计值得保留。

一段“已经修改完成”的聊天回复不是 Artifact；被保存、定位、版本化并交给下游使用的文件或数据才是。

### 10. Claim：系统正在声称什么事实

Evidence 经常被直接建成附件列表，却没有说明它证明什么。缺少 `Claim`，Evidence 就只是材料堆积。

Claim 是一个可验证断言，例如：

```text
“测试通过”
“版本已经推送到远端”
“生产接口返回预期结果”
“这段回答来自当前有效的制度文档”
```

Claim 需要作用对象、范围、Revision、时间和声明主体。同一句“发布成功”，在测试环境和生产环境是两个不同 Claim。

### 11. Evidence：支撑或反驳 Claim 的可验证事实

`Evidence` 属于 Trust & Verification 领域。它把 Claim 与 Source、Run、Artifact、Observation、Verifier 和时间连接起来。

一条合格 Evidence 至少需要：

```json
{
  "claimId": "claim_production_ready",
  "sourceRef": "check://production/health",
  "producedByRunId": "run_20260802_01",
  "observedAt": "2026-08-02T17:00:00+08:00",
  "revision": "release_42",
  "integrity": "sha256:...",
  "verifier": "health-check-v3"
}
```

Trace、日志、截图和测试输出只能算 Evidence Candidate。只有当它被绑定到具体 Claim，并说明来源、范围和验证方式后，才成为可以用于验收的 Evidence。

### 12. Evaluation：依据标准作出的判断

`Evaluation` 消费 Artifact、Claim 和 Evidence，根据 Criteria 给出 Verdict、Score、Explanation 或 Risk。

```text
Evidence：生产接口在 17:00 返回 200
Criteria：连续三次健康检查成功，响应体版本为 release_42
Evaluation：PASS / FAIL / INCONCLUSIVE
```

Anthropic 的 Evaluator-optimizer 模式和 OpenAI Agents SDK 的 Guardrail / Eval 能力都说明“生成”和“判断”需要分离。Evidence 是事实材料，Evaluation 是基于规则或评审者作出的判断，两者不能混成一个分数字段。

## 哪些对象需要按场景升级为一级

除了十二个核心对象，还有一组条件性对象。是否独立建模，取决于业务复杂度。

| 候选对象 | 什么时候升级为一级对象 | 否则放在哪里 |
| --- | --- | --- |
| Memory | 跨 Run 保存、可纠错、可删除、需要来源与保留策略 | Context 的引用或 Session State |
| Approval | 运行会跨小时或跨天暂停，需要记录批准人、范围与失效条件 | Run Event / Policy Decision |
| Workspace | 有独立权限、Revision、文件锁、隔离和生命周期 | Context Scope |
| Handoff / Delegation | 多 Agent 间需要追踪责任转移、输入输出契约和超时 | Task 的关系事件 |
| Trigger / Schedule | 周期任务、Webhook、事件订阅需要独立启停和重放 | Task 创建来源 |
| Budget / Usage | 存在额度分配、成本归属、限额和结算 | Run Metric |
| Version / Release | Agent 配置需要灰度、回滚、对比评测和生产追踪 | Agent 的版本字段 |
| Receipt | 需要对外签发、不可变、可撤销或具备法律/业务效力 | Claim、Evidence、Evaluation 的聚合视图 |

例如 [OpenAI Agents SDK 的 Human-in-the-loop](https://openai.github.io/openai-agents-python/human_in_the_loop/)允许 Run 因 Tool Approval 暂停、序列化并在之后恢复。在短对话里，Approval 可以只是一次事件；在跨天的财务或发布流程里，它需要自己的 ID、审批人、Decision、Scope、Policy Version 和 Expiry，此时就应该成为一级对象。

## 哪些常见名词默认不该升为一级

### Prompt

Prompt 是 Agent Version 或 Capability Implementation 的组成部分。只有当平台专门做 Prompt 资产管理、评测、发布和回滚时，它才需要独立生命周期。

### Model 与 Runtime

它们是可替换执行依赖。模型路由平台会把 Model Deployment 当一级对象，通用 Agent 产品则应避免让业务 Task 直接绑定供应商模型。

### Tool 与 Skill

Tool 是调用接口，Skill 是完成任务的方法。它们可以作为 Capability 的实现对象独立管理，但业务路由首先应该依赖 Capability，而不是某个函数名或 Skill 文件名。

### Message

Message 是通信协议对象，不等于 Task、Context 或 Artifact。一串消息可以产生多个 Task，一个长 Task 也可以跨越多轮 Message。

### Event、Log 与 Trace

它们记录运行轨迹，主要属于 Observability。OpenAI Agents SDK 的 Trace 会记录 Model Generation、Tool Call、Handoff 和 Guardrail 等 Span，这对排障很重要，但 Trace 不自动证明业务结果成立。

### Workflow 与 Graph

Workflow 是编排定义，Graph 是一种结构表达。以 Workflow 为产品核心的平台可以把它作为一级对象；以开放式 Agent 为核心的系统，更适合把它作为 Capability 或 Execution Plan 的实现。

## 一套可落地的最小关系模型

如果从数据库和 API 开始，我会先建立下面几条稳定关系，而不是先画复杂的多 Agent 拓扑：

```text
Agent --has--> AgentVersion
Identity --authorizes--> Agent
Goal --decomposes_into--> Task
Agent --assigned_to--> Task
Task --attempted_by--> Run
Run --uses--> Capability
Run --reads--> ContextSnapshot
Policy --governs--> Run
Run --produces--> Artifact
Run --asserts--> Claim
Evidence --supports_or_refutes--> Claim
Evaluation --judges--> Claim
```

对应的生命周期应明确区分：

```text
Task:  created → assigned → active → completed / failed / canceled
Run:   queued → running → waiting_approval → paused → completed / failed / interrupted
Claim: proposed → supported / refuted / insufficient
Eval:  pending → pass / fail / inconclusive
```

这里最重要的不是状态名称，而是不要用一个 `status = done` 同时代表“Agent 停止运行”“文件已经生成”“测试通过”“代码已经发布”“业务已经验收”。这些是不同对象上的不同 Claim。

## 应该按什么顺序建设

### 第一阶段：先让工作可运行

最小闭环需要：

```text
Agent + Task + Run + Capability + Context + Artifact
```

这一阶段解决“谁在什么上下文里，用什么能力，完成了哪项工作，产出了什么”。

### 第二阶段：让结果可信

进入真实业务前补齐：

```text
Identity + Policy + Claim + Evidence + Evaluation
```

这一阶段解决权限、责任、结果边界与验真。系统只有 Trace，没有 Claim 和 Evidence Model，仍然无法可靠验收。

### 第三阶段：处理长期与协作

当任务跨 Session、跨 Agent、跨天运行时，再把以下对象提升出来：

```text
Memory + Approval + Workspace + Handoff + Trigger + Budget + Version
```

它们不应该为了架构图完整而提前建设。升级信号应来自真实问题：上下文无法恢复、责任转移说不清、审批失效、成本无法归属、版本无法回滚。

## 我的判断框架

遇到一个新的 Agent 名词，我会按下面的顺序判断：

1. 它描述的是业务世界，还是某个 SDK 的实现方式？
2. 它是否拥有独立 ID、生命周期和业务规则？
3. 多个 Task、Run 或 Agent 是否需要共同引用它？
4. 更换 Model、Runtime、Protocol 后，它是否仍然成立？
5. 不独立建模会不会导致权限、恢复、审计或验收说不清？

五个问题大多回答“是”，就把它提升为一级领域对象；否则先作为属性、事件、协议对象或实现对象存在。

这套模型给我的最大帮助，是把 Agent Platform 从“模型、Prompt、Tool 的配置集合”重新理解成责任系统：Agent 承担工作，Run 改变环境，Artifact 承载结果，Claim 表达结论，Evidence 与 Evaluation 决定我们能否相信它。

真正进入生产后，模型能力会持续变化，Runtime 也可以替换。长期稳定的资产，是这些领域对象及其关系。
