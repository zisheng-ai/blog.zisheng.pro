---
title: GBrain、Sirchmunk、Understand Anything 怎么搭配：Agent 知识基础设施的三层实践
date: 2026-07-22 02:00:00
description: GBrain、Sirchmunk 和 Understand Anything 都在解决 Agent 的上下文问题，却分别面向长期知识、实时原始数据和代码结构。本文结合桌面 Agent Host 的架构实践，拆解三者的能力边界、组合方式、云端与本地部署，并给出从单一知识底座逐步演进的选型框架。
categories:
  - [AI]
tags:
  - GBrain
  - Sirchmunk
  - Understand Anything
  - RAG
  - Knowledge Graph
  - Agent Memory
  - AI Infra
cover: /images/gbrain-sirchmunk-understand-anything.webp
---

最近在设计一个 Agent Host 的知识架构时，我连续遇到了三个名字：GBrain、Sirchmunk 和 Understand Anything。

它们都宣称能让 Agent 更懂知识、更懂文件或更懂代码，功能列表里也都有 Search、Knowledge、Graph、MCP。第一眼很容易把它们排成一条升级路线：普通 RAG 不够，就上 Graph；Graph 还不够，就做 LLM Wiki；最后找一个“大而全”的项目把其他方案全部替代。

但我真正把场景拆开后发现，这个思路的问题很大。团队维护的 Spec、正在变化的本地 Diff、函数之间的调用关系，本来就是三种不同的数据。硬塞进同一个检索范式，最终往往不是架构统一，而是让一个组件承担它不擅长的工作。

这篇文章记录我目前的判断：三者分别适合什么，什么时候组合，以及为什么第一版不应该一次性把它们全部接入。

## 一句话总结

**GBrain 负责沉淀长期知识，Sirchmunk 负责探索实时原始数据，Understand Anything 负责理解代码结构。它们不是相互替代的升级关系，而是 Knowledge Plane、Raw Context Search 和 Code Intelligence 三个不同层次。**

如果只能先选一个，我会以 GBrain 作为知识底座；Sirchmunk 和 Understand Anything 都等真实瓶颈出现后，再作为 Adapter 接入。

## 先把三个项目放回各自的问题里

| 项目 | 核心问题 | 最适合的数据 | 典型输出 |
|---|---|---|---|
| GBrain | 怎样让知识长期积累、关联并被 Agent 稳定使用 | Wiki、Spec、ADR、研究结论、团队知识 | 页面、Citation、Hybrid Search、Graph、综合回答 |
| Sirchmunk | 怎样直接从快速变化的原始数据里找到有效证据 | 本地文件、日志、临时文档、未提交内容 | Evidence Snippet、Knowledge Cluster、实时回答 |
| Understand Anything | 怎样理解代码库的结构和影响范围 | Repository、函数、类、Import、Call、Diff | Code Graph、Domain Flow、Diff Impact、Guided Tour |

三个项目都能“搜索”，但搜索对象和目标不同。

搜索一篇 ADR 的结论，核心是语义召回和 Citation；搜索刚生成的构建日志，核心是不用等待重新建索引；回答“修改这个接口会影响哪些模块”，核心是调用和依赖关系。把它们都叫 RAG，会掩盖真正的工程差异。

## GBrain：团队和 Agent 的长期知识底座

[GBrain](https://github.com/garrytan/gbrain) 更接近一个完整的 Agent Brain：Markdown Brain Repo 保存长期知识，检索层组合 Keyword、Vector、RRF 和 Graph Signal，查询层既能返回原始结果，也能生成带 Citation 的综合回答和知识缺口分析。

它同时提供两种有吸引力的部署形态：

| 场景 | 引擎 |
|---|---|
| 个人、本地、零服务部署 | PGLite |
| 团队共享、大规模、多机器 | Postgres + pgvector |

这对桌面 Agent 产品很合适：云端可以维护团队 Brain，本地可以维护当前 Workspace 的 Brain，Agent 在断网时仍然能查询已经同步的知识。

更重要的是，GBrain 覆盖了我最初理解的 LLM Wiki 核心能力：知识不是永远停留在原始 Chunk，而是被编译为可读、可链接、可持续维护的页面。查询产生的新结论也可以继续积累，而不是随着 Chat Session 消失。

不过这里有一个关键边界：**GBrain 是知识引擎，不应该自动成为业务系统的权限和发布中心。**

在企业或团队场景里，我仍然需要外层系统负责：

- 哪个 Knowledge Version 已经发布；
- 当前 Workspace 能访问哪些知识；
- 哪些页面经过 Owner 审核；
- 哪个 Session 固定了哪个 Revision；
- 查询和执行是否需要进入 Audit。

因此，我更愿意把 GBrain 放在 Knowledge Plane，把版本、签名、权限和审计留给宿主系统。

## Sirchmunk：不预建索引，直接探索 Raw Data

[Sirchmunk](https://github.com/modelscope/sirchmunk) 的价值不在于又实现了一套 Vector RAG，而是它选择了另一条路径：直接面对 Raw Data 做 Indexless Agentic Search。

它通过 Grep Retriever、关键词级联和 Monte Carlo Evidence Sampling，从原始文件里定位相关区域，再让 LLM 综合为 Knowledge Cluster。官方提供 FAST、DEEP 和 FILENAME_ONLY 等模式，也提供 MCP，适合直接挂到 Agent 上。

这个设计特别适合快速变化的数据：

```text
未提交 Diff
临时日志
构建输出
诊断文件
刚下载的大型原始资料
```

这些内容如果每改一次就重新 Embedding、重新构图，维护成本可能比检索收益还高。Sirchmunk 的思路是先搜索当前真相，再决定哪些结论值得沉淀。

但它不适合直接接管正式团队知识。一次 Agentic Search 得到的 Knowledge Cluster，仍然需要回答：来源是什么、是否过期、是否和正式 Spec 冲突、谁审核过。我的处理方式是把 Sirchmunk 的结果当作当前任务上下文；需要长期复用时，先转成 Knowledge Draft，再进入审核和发布。

## Understand Anything：代码不是普通文档

[Understand Anything](https://github.com/Lum1104/Understand-Anything) 解决的是另一类问题：Agent 能读到代码，却缺少整个 Repository 的结构认识。

它使用 Tree-sitter 和多 Agent Pipeline 提取 File、Function、Class、Dependency 等节点与关系，生成可交互 Knowledge Graph，并提供 Architecture Layer、Domain Flow、Diff Impact、Guided Tour 等视图。它还能分析 Karpathy 风格的 LLM Wiki，把显式 Wikilink 和隐式知识关系可视化。

这和把源代码切 Chunk 做 Vector Search 有本质区别。

假设我问：

> 修改 `PermissionBroker` 的返回结构，会影响哪些 Workflow、Adapter 和 UI？

向量检索能找到“语义相似”的文件，但它不能保证遍历真实 Import、Call 和 Type Dependency。Code Graph 的价值就在于把这类结构事实显式化。

Understand Anything 的输出也不应该被理解为第二份代码事实源。代码仍然以 Git Revision 为准，Graph 是可重建 Projection。只要这一点守住，它很适合作为 Repository Knowledge 的生产器，把代码结构结果继续交给 GBrain 做长期查询和关联。

## 正确组合：一个底座，两个按需 Adapter

我目前采用的组合关系是：

```text
团队 Spec / ADR / Workflow ────────┐
                                   │
Repository Revision ─ Understand Anything
                                   │
                                   ▼
                              GBrain Cloud
                                   │
                         发布版本化 Knowledge
                                   │
                                   ▼
                              GBrain Local
                                   ▲
当前 Workspace / Diff ─ Sirchmunk ┘
                                   │
                                   ▼
                              Coding Agent
```

但这张图容易产生一个误解：Sirchmunk 的结果不是默认全部写入 GBrain。更准确的运行逻辑是：

1. 正式知识优先查询 GBrain；
2. 问题涉及当前未提交内容时，查询 Sirchmunk；
3. 问题涉及调用链和 Diff Impact 时，查询 Understand Anything；
4. 多个结果由宿主系统合并，并保留各自 Citation；
5. 只有经过验证和审核的稳定结论，才进入 GBrain 的正式页面。

统一接口可以很小：

```ts
type ContextIntent =
  | "knowledge"
  | "raw-files"
  | "code-graph"
  | "auto";

interface ContextQuery {
  workspace: string;
  revision: string;
  query: string;
  intent: ContextIntent;
}

interface ContextResult {
  engine: "gbrain" | "sirchmunk" | "understand-anything";
  content: string;
  citation: string;
  digest: string;
}
```

Agent 不需要知道三套引擎的数据库、进程和权限模型，只需要调用宿主系统暴露的 `knowledge.search`、`workspace.search` 和 `code.graph_query`。

## 云端和本地应该怎样分工

我不赞成“所有知识都在云端”或“所有知识都只在本地”这两种极端方案。

云端适合维护：

- 团队 Spec、ADR、Workflow 和 Skill；
- 已发布 Repository Revision 的稳定分析结果；
- Owner Review 后的 Wiki 页面；
- 跨团队共享检索和正式 Knowledge Version。

本地适合维护：

- 当前 Workspace；
- 未提交 Diff；
- 本地 Overlay；
- 当前 Session 固定的 Knowledge Revision；
- 不能默认上传的代码、日志和执行结果。

同步的也不应该是数据库文件，而应该是版本化资产：

```text
Cloud Knowledge
  → Asset Version + Digest + Signature
  → Local Verify
  → Materialize Local Brain
  → Build Index
  → Atomic Activate Revision
```

这样才能做到离线、回滚和可复现。如果 Agent 每次都读云端的“最新知识”，同一个长 Session 前后可能得到不同结论，审计和复现都会失去基础。

## Execution Evidence 不等于 Wiki Citation

在讨论这套系统时，我反复看到 Evidence 这个词，但它不是 LLM Wiki 独有的核心概念。

| 概念 | 含义 |
|---|---|
| Source | 原始材料 |
| Citation | 结论引用的具体位置 |
| Provenance | 知识经历了哪些加工和审核 |
| Execution Evidence | Agent 的动作是否真的成功 |

例如“测试已经通过”只是一句 Agent Message；真正的 Execution Evidence 应该包含命令、退出码、测试数量、时间和对应 Revision。

Evidence 可以反哺知识，但不能自动成为知识：

```text
测试结果 / Diff / Commit
        ↓
Execution Evidence
        ↓
提炼稳定机制
        ↓
Knowledge Draft
        ↓
Review + Citation
        ↓
正式 Wiki
```

这个边界非常重要。否则一次错误推理或偶发现象写进团队 Wiki，之后所有 Agent 都会把它当成事实。

## 第一版不要三件套全上

从工程投入看，三者同时接入意味着：

- 三套生命周期和升级策略；
- 三套索引或持久化；
- 多种查询结果的融合与去重；
- 更多 API Key、Sidecar 和供应链风险；
- 更复杂的 Revision、Permission 和 Citation 映射。

因此我的实施顺序是：

| 阶段 | 目标 |
|---|---|
| Phase 1 | GBrain Local 跑通团队 Knowledge 到 Agent 的查询闭环 |
| Phase 2 | 增加 GBrain Cloud、Knowledge Draft、Review 和版本发布 |
| Phase 3 | Raw Files 实时检索成为瓶颈时接 Sirchmunk |
| Phase 4 | 调用链、依赖和 Diff Impact 成为瓶颈时接 Understand Anything |

甚至在小型 Repository 中，`rg + LSP + 架构文档` 可能已经够用。是否引入新组件，应该由真实任务的 Recall、Precision、Latency、Token 和维护成本决定，而不是由功能列表决定。

## 要不要用 / 我的判断框架

我的结论不是“三个都值得上”，而是分三档：

### 值得上：GBrain

如果 Agent 需要跨 Session 积累团队知识，需要云端共享、本地离线和长期 Citation，GBrain 值得作为 Knowledge Plane 验证。它覆盖了 LLM Wiki、Hybrid RAG、Graph 和 Synthesis，能先建立一条完整主链路。

### 可以再等：Sirchmunk

如果当前 `rg`、FTS 或 GBrain 对 Workspace Raw Files 已经够快，Sirchmunk 可以先不接。等频繁重建索引、临时大文件和日志检索真的影响体验，再引入它的 Indexless Search。

### 重点关注：Understand Anything

如果产品的核心价值包含 Repository Onboarding、影响分析和架构理解，Understand Anything 很有价值；如果只是让 Coding Agent 修改几个文件，它可能暂时是过度设计。

最终我用一句话约束自己的选型：

> **先建立一个稳定知识底座，再用真实任务证明是否需要实时搜索器和代码图谱；不要为了“看起来完整”一次性维护三套基础设施。**

这也是我从前端工程走向 Agent / AI Infra 后越来越强烈的体会：真正的架构能力，不是把最多的组件画进图里，而是知道每个组件何时应该缺席。
