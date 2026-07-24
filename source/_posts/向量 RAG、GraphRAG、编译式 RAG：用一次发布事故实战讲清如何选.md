---
title: 向量 RAG、GraphRAG、编译式 RAG：用一次发布事故实战讲清如何选
date: 2026-07-24 14:00:00
description: 向量 RAG、GraphRAG 和编译式 RAG 并不是一条从低级到高级的升级路线。本文用一个发布事故知识助手贯穿三种方案，拆解数据如何进入系统、问题如何命中证据、失败会发生在哪里，并给出从 MVP 到生产架构的选型矩阵、评测方法与组合策略。
categories:
  - [AI]
tags:
  - RAG
  - Vector Search
  - GraphRAG
  - LLM Wiki
  - Knowledge Compilation
  - AI Infra
cover: /images/rag-three-routes-decision.webp
---

![向量 RAG、GraphRAG、编译式 RAG 选型图](/images/rag-three-routes-decision.webp)

*先看问题形状，再决定知识该被表示成 Chunk、Graph，还是可持续维护的编译产物。*

最近看到一张图，把知识检索分成向量 RAG、GraphRAG 和编译式 RAG 三条路线。图很直观，但也留下了最难回答的部分：一个真实系统到底怎么选？

如果只按概念解释，很容易得到三句没法指导架构的结论：小资料库用向量，复杂关系用 GraphRAG，可维护交付用编译式 RAG。落地时，资料库多大算小、什么关系值得建图、编译出来的知识丢了事实怎么办，仍然没有答案。

这篇文章不再横向罗列名词。我会用同一个“发布事故知识助手”贯穿三条路线，观察它们如何处理同一批文档、回答三类问题，再给出我会如何从 MVP 演进到生产架构。

## 一句话总结

我的选择顺序是：

1. **默认先做 Hybrid RAG**：Keyword / BM25 保精确词，Vector Search 处理自然语言表达，再加 Metadata Filter 与 Rerank。
2. **当问题依赖稳定的实体关系和多跳路径时，增加 GraphRAG**：它是一个面向关系问题的专用索引，不是默认底座。
3. **当团队反复综合同一批稳定知识，并希望结果能够积累、审阅和复用时，增加编译层**：把原始资料加工成结构化 Wiki、Procedure、Decision 和 Conflict 页面，但原始 Source 仍然是事实源。

生产系统往往不是三选一，而是：

```text
Raw Sources
  ├─ Hybrid Index        → 找相关原文
  ├─ Relation Graph      → 走依赖与影响路径
  └─ Compiled Knowledge  → 复用已经整理过的知识
                ↓
        Evidence Gateway
                ↓
          LLM / Agent
```

三条路线共享 Scope、Revision、Permission、Citation 和 Trace。不要让 Agent 直接面对三个互不兼容的知识后端。

## 先把“编译式 RAG”这个词说准确

Vector RAG 和 GraphRAG 已经有相对明确的工程含义，“编译式 RAG”却还不是一个边界稳定的标准术语。不同人可能用它指：

- 提前生成摘要或 Contextual Chunk；
- 把整个知识库放进长上下文并缓存 KV；
- 把原始资料编译成相互链接的 Markdown Wiki；
- 从资料中抽取可执行的 Agent Skill；
- 把多种检索计划编译成一个查询。

本文采用其中最适合知识工程的一种定义：

> 编译式 RAG，是把不可直接高效消费的 Raw Sources，在构建阶段加工成面向后续任务的、可检查、可版本化、可增量更新的知识中间表示；查询时优先读取这些编译产物，必要时回到原文核验。

这个定义主要对应 [Andrej Karpathy 的 LLM Wiki pattern](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f)：原始资料保持不可变，LLM 增量维护一个互相链接的 Wiki，再用 Schema / Agent Instruction 约束写入和查询流程。

“编译”这个比喻有四个关键点：

| 编译器概念 | 知识系统里的对应物 |
| --- | --- |
| Source Code | PDF、Runbook、ADR、代码、事故复盘等原始事实源 |
| Intermediate Representation | Topic Page、Entity Page、Procedure、Decision、Conflict、Citation Map |
| Compiler | LLM + 确定性解析器 + Schema + 校验器 |
| Build Artifact | 可被人阅读、被 Agent 搜索和遍历的 Wiki / Knowledge Pack |

它不是“预先做了 Embedding”的新名字。Embedding 只是建立向量索引；编译式 RAG 会合并跨文档事实、建立链接、显式记录冲突，甚至把陈述性知识整理成可执行流程。

它也不能自动取代 RAG。编译过程会丢信息、写错关系、把条件句压成绝对规则。2026 年的 [WiCER 研究](https://arxiv.org/abs/2605.07068)专门讨论了这个 compilation gap：盲目压缩原始知识会出现严重信息损失，必须用诊断问题找出丢失事实，再迭代修正编译产物。

## 实战场景：给发布系统做一个知识助手

假设团队有八份资料：

| ID | 文档 | 主要内容 |
| --- | --- | --- |
| D1 | `service-topology.md` | 服务、数据库、消息队列和用户链路 |
| D2 | `release-runbook.md` | 发布步骤、观察窗口、回滚入口 |
| D3 | `feature-flag-policy.md` | 灰度比例、开关 Owner、自动熔断阈值 |
| D4 | `change-184.md` | 本次接口升级涉及的服务与版本 |
| D5 | `incident-042.md` | 一次灰度发布导致结算失败的复盘 |
| D6 | `dependency-registry.yaml` | 服务间调用与数据依赖 |
| D7 | `rollback-checklist.md` | 回滚前后要检查的指标 |
| D8 | `exceptions.md` | 大促、数据迁移期间的例外规则 |

为了避免把 Demo 做成“上传 PDF，然后问一句摘要”，我选三类形状完全不同的问题：

```text
Q1 事实查找：
结算服务发布失败后，默认观察窗口是多久？

Q2 关系与影响：
把 checkout-api 从 v2 升到 v3，会影响哪些用户链路？为什么？

Q3 稳定执行：
灰度期间错误率持续上升，Agent 应该按什么条件判断、执行和验证回滚？
```

这三问分别要求系统找到一个事实、走一条关系链、复用一套跨文档流程。它们比“总结这些文档”更接近真实系统。

## 第一轮：用向量 RAG 解决

### 数据怎样进入系统

最小 Vector RAG 会做四件事：

```text
Document
  → Parse
  → Chunk
  → Embedding
  → Vector Index
```

生产环境我不会只留向量通道。错误码、服务名、版本号和配置项更适合 BM25，因此实际链路会是：

```text
Query
  ├─ Keyword / BM25
  ├─ Vector Search
  └─ Metadata Filter
          ↓
      Rank Fusion
          ↓
       Reranker
          ↓
   Evidence Top K
```

每个 Chunk 至少要保留：

```json
{
  "chunkId": "release-runbook#rollback-window",
  "content": "默认观察 10 分钟；错误率连续 3 分钟超过 2% 时进入回滚判断。",
  "sourceUri": "kb://runbooks/release",
  "sourceVersion": "sha256:...",
  "section": "回滚条件",
  "scope": ["production"],
  "validFrom": "2026-06-01",
  "permission": "engineering"
}
```

没有这些 Metadata，检索器可能把旧版本、测试环境或无权限文档排在最前面。相似度高，不等于业务上可用。

### 三个问题会发生什么

Q1 很适合 Vector / Hybrid RAG。Query 中的“发布失败”“观察窗口”和 D2 的表述足够接近，Top K 很容易命中正确段落。若用户直接搜 `rollback_window`，BM25 通道还能兜住精确标识符。

Q2 开始吃力。向量检索可能召回 `change-184.md` 和 `service-topology.md`，但它不会天然执行这条路径：

```text
checkout-api v3
  → 修改 price contract
  → cart-service 消费该 contract
  → Web Checkout 与 Mobile Checkout 依赖 cart-service
```

Top K 里只要漏掉一段，影响分析就断了。把 `k` 调大能提高 Recall，却会带来更多无关上下文和更高推理成本。

Q3 的问题更明显。回滚判断散落在 D2、D3、D7 和 D8：阈值在策略文档，动作在 Runbook，验证项在 Checklist，大促例外又覆盖默认规则。单轮 RAG 每次都要重新找到四份资料并现场拼装流程。

### 向量 RAG 的真实边界

Vector RAG 擅长的是“从大语料里找到语义相近的局部证据”。它没有承诺：

- 一次检索覆盖完整依赖链；
- 不同 Chunk 之间的方向关系正确；
- 每次都以相同顺序重建同一套流程；
- 自动识别新规则覆盖旧规则；
- 将本次回答沉淀为下一次可复用的知识。

所以它适合作为默认检索底座，却不等于完整知识架构。

## 第二轮：什么时候 GraphRAG 值得上

GraphRAG 在构建阶段把文本转成实体和关系：

```text
(checkout-api:v3)
  -[CHANGES_CONTRACT]-> (price-contract:v3)

(cart-service)
  -[CONSUMES]-> (price-contract)

(web-checkout)
  -[DEPENDS_ON]-> (cart-service)
```

查询 Q2 时，系统先定位 `checkout-api:v3`，再按允许的 Edge Type 遍历，最后把路径关联的原文片段一起交给模型。此时“为什么受影响”不再靠模型从若干相似 Chunk 中猜，而是有一条显式路径。

微软的 [GraphRAG 官方索引流程](https://microsoft.github.io/graphrag/index/overview/)会从原文抽取 Entity、Relationship 和 Claim，做 Community Detection、生成 Community Report，并保留向量表示。它的 [Local Search](https://microsoft.github.io/graphrag/query/local_search/)会结合实体关系与原始 Text Unit，适合具体实体问题；Global Search 则基于 Community Report 做 Map-Reduce，更适合“整批资料有哪些主要主题”这类全局问题。

### GraphRAG 不是“Vector RAG 加一个图数据库”

难点不在于把 Node 和 Edge 存进去，而在于：

1. **实体消歧**：`checkout`、`checkout-api` 和“结算接口”是不是同一个实体？
2. **关系方向**：A 调用 B 与 B 依赖 A 不能写反。
3. **时间有效性**：v2 的关系不能污染 v3。
4. **来源追踪**：每条 Edge 必须能回到产生它的 Source Span。
5. **增量更新**：一次 Change 合并后，哪些社区摘要和关系要重算？
6. **权限裁剪**：删掉无权访问的 Node 后，剩余路径是否还泄露信息？

如果团队不能维护这些约束，GraphRAG 会把“检索偶尔漏片段”变成“图谱稳定地提供错误关系”。

### 我会怎样限制第一版图谱

不从所有自然语言里自由抽取任意关系，而是先定义少量 Typed Edge：

```ts
type EdgeType =
  | 'CALLS'
  | 'READS_FROM'
  | 'WRITES_TO'
  | 'DEPENDS_ON'
  | 'OWNED_BY'
  | 'SUPERSEDES'
  | 'MITIGATED_BY';
```

结构化配置和代码依赖用确定性解析器产生；只有事故复盘、会议纪要等非结构化资料才交给 LLM 抽取，并标记较低 Confidence，等待审核。

这样做牺牲了“自动构建万物图谱”的想象力，换来可验证的影响分析。

## 第三轮：编译式 RAG 改变了什么

Q3 是编译式 RAG 最有价值的地方。团队已经反复问过“什么条件触发回滚”“回滚后验证什么”，系统没必要每次从四份原文重新拼一遍。

构建阶段可以产出：

```text
wiki/
  systems/
    checkout-api.md
    cart-service.md
  procedures/
    canary-rollback.md
  decisions/
    change-184.md
  incidents/
    incident-042.md
  conflicts/
    release-exceptions.md
  index.md
  log.md
```

其中 `procedures/canary-rollback.md` 不是一段随意摘要，而是带来源和适用条件的结构化产物：

```yaml
---
type: procedure
status: reviewed
revision: 7
applies_to:
  environment: production
  excluded_windows:
    - major_campaign
source_refs:
  - kb://runbooks/release#rollback
  - kb://policies/feature-flag#threshold
  - kb://checklists/rollback#verification
compiled_from:
  - sha256:...
  - sha256:...
---
```

正文再写清：

```text
触发条件
1. 灰度错误率连续 3 分钟超过 2%；
2. 确认不是监控延迟；
3. 当前不处于已批准的例外窗口。

动作
1. 冻结继续放量；
2. 关闭 v3 Feature Flag；
3. 恢复 v2 流量；
4. 记录 Change 与执行人。

验证
1. 错误率回到基线；
2. 核心链路通过；
3. 无积压消息与脏数据；
4. Evidence 写回本次 Run。
```

Agent 查询时先读 `index.md`，找到 Procedure，再沿链接读取相关 System、Decision 和 Conflict 页面。对于关键阈值，它仍要回到 Source Span 核验。

### 编译式 RAG 的收益不是“少搜一次”

它把重复劳动从 Query Time 移到 Build Time：

- 跨文档合并提前完成；
- 常用概念拥有稳定页面；
- 冲突与覆盖关系被显式记录；
- Procedure 可以进入 Review 和版本管理；
- 上一次探索产生的有效结构能够被下一次复用。

这更接近程序编译：开发者不会每次运行程序时都重新理解所有 Source Code，而是消费经过检查的 Artifact。

### 编译产物绝不能成为未经审查的新事实源

编译器会出错，LLM Compiler 也会。至少要守住四条线：

1. Raw Sources 不可被编译器改写。
2. 每个 Claim 都能追到 Source Span 与 Source Revision。
3. 编译产物可以全部删除并重建。
4. 高风险 Procedure 在进入 `reviewed` 前不能驱动自动执行。

如果一个 Wiki 页面引用另一个 Wiki 页面，最后却找不到原文，这会形成循环证据。页面之间可以互相导航，事实依据必须落回 Source。

## 把三条路线放回同一张运行图

![向量 RAG、GraphRAG、编译式 RAG 的构建时与查询时成本](/images/rag-three-routes-runtime.webp)

*三条路线都在“预处理”，区别在于构建出了什么中间表示，以及查询时还要做多少工作。*

向量 RAG 的构建成本最低，查询时要重新召回和综合。GraphRAG 把实体、关系与社区摘要前移到构建阶段，换来更可控的多跳与全局查询。编译式 RAG 进一步把常用综合结果和流程前移，但也承担最大的知识维护与编译正确性责任。

生成的直觉图更容易看出这三个动作的差别：

![从语义检索、关系网络到知识编译的三种工作方式](/images/rag-three-routes-workshop.webp)

*向量路线在空间中找邻近片段；图路线沿关系走路径；编译路线把原始资料加工成可长期使用的知识产物。*

## 桌面推演：三问分别应该命中什么

这不是一个声称“准确率提升 37%”的伪 Benchmark。数据量太小、问题由作者设计、没有盲测，百分比没有意义。这里做的是架构桌面推演：先写出每个问题必须命中的证据与关系，再看哪条路线最自然。

| Query | 正确答案所需材料 | Vector / Hybrid | GraphRAG | 编译式 RAG |
| --- | --- | --- | --- | --- |
| Q1 观察窗口多久 | D2 的一个段落 | 最自然 | 可以，但过度设计 | 可以，仍应核验原文 |
| Q2 升级影响哪些链路 | D4 + D6 + D1 的依赖路径 | 容易漏一跳 | 最自然 | 若已有 System Pages 也能做 |
| Q3 如何判断并执行回滚 | D2 + D3 + D7 + D8 | 每次重新拼装 | 图能找关联，但流程顺序仍需组织 | 最自然，Procedure 可审阅复用 |

还要加入反向问题：

```text
Q4：灰度错误率 1.5% 时是否必须回滚？
```

正确行为不是从“错误率上升”联想到回滚，而是根据阈值明确回答“不满足默认自动回滚条件”，并指出是否存在其他人工判断规则。

再加入冲突问题：

```text
Q5：大促窗口能否自动关闭 Feature Flag？
```

系统必须同时找到默认策略和 D8 的例外规则，按有效 Revision 判断谁覆盖谁。如果它只返回默认 Runbook，即使语言很流畅也算失败。

## 评测不能只看最终回答

三条路线需要不同的中间指标。

### Vector / Hybrid RAG

| 指标 | 要回答的问题 |
| --- | --- |
| Recall@K | 必要 Source Span 是否进入候选集 |
| MRR / NDCG | 最有用证据是否排在前面 |
| Filter Accuracy | Scope、时间、权限是否过滤正确 |
| Citation Precision | 引用是否真的支持对应 Claim |
| No-evidence Refusal | 没有证据时是否拒答 |

### GraphRAG

| 指标 | 要回答的问题 |
| --- | --- |
| Entity Resolution Accuracy | 同名、别名、版本实体是否合并正确 |
| Edge Precision / Recall | 关系是否存在、方向是否正确 |
| Path Coverage | 关键影响路径是否完整 |
| Temporal Validity | 查询时是否使用了正确版本的关系 |
| Source Traceability | Node / Edge 是否能回到原文 |

### 编译式 RAG

| 指标 | 要回答的问题 |
| --- | --- |
| Fact Preservation | 编译后是否丢掉关键条件与数字 |
| Claim-Citation Alignment | 页面中的每个 Claim 是否被引用支撑 |
| Conflict Detection | 新旧规则冲突是否被标出 |
| Rebuild Determinism | 相同 Source 与 Compiler Version 是否得到可解释的 Diff |
| Staleness | Source 变化后，受影响页面多久完成重编译 |
| Procedure Success Rate | Agent 按编译流程执行是否真的完成任务 |

一个系统最终答对了，不代表索引正确。LLM 可能靠训练知识猜中答案。Eval 数据里要包含未公开的内部值、旧版本干扰项和“应该拒答”的问题，才能测出 Grounding。

建议把用例写成可执行契约：

```json
{
  "id": "release-q3",
  "query": "灰度错误率持续上升，应该如何回滚？",
  "scope": {
    "environment": "production",
    "revision": "release-2026-07"
  },
  "expectedEvidence": [
    "release-runbook#rollback",
    "feature-flag-policy#threshold",
    "rollback-checklist#verification",
    "exceptions#campaign-window"
  ],
  "mustMention": [
    "连续 3 分钟",
    "冻结放量",
    "验证错误率回到基线"
  ],
  "mustNotDo": [
    "在例外窗口未经审批自动关闭开关"
  ]
}
```

## 选型时，我看这六个变量

### 1. 问题是找片段、走关系，还是复用综合结果

这是第一判断，优先级高于语料规模。

- “这条规则写在哪里”是 Lookup。
- “改了它会影响谁”是 Relation。
- “以后每次都该怎么做”是 Reuse / Procedure。

问题形状判断错了，后面调 Embedding Model、Graph Database 或 Prompt 都只是优化错误架构。

### 2. 关系是否稳定

组织架构、服务依赖、法规条款之间的关系相对稳定，值得建图。聊天记录里临时提到的“可能相关”，通常不值得成为长期 Edge。

关系变化比查询还快时，GraphRAG 的增量维护成本会吞掉收益。

### 3. Query 是否重复

编译的收益依赖复用。同一批资料只问一次，直接 RAG 通常更便宜；团队每周都要重新综合同一套规则，提前编译才开始划算。

这里不能只算 Token。还要算人工复核、错误传播和旧知识清理成本。

### 4. Source 是否持续变化

变化快的日志、当前代码和临时文件，优先走实时 Search。稳定的 ADR、正式 Runbook、领域概念和高频 Procedure，更适合编译。

一个常见组合是：

```text
快速变化层：Hybrid Search 直查当前 Source
稳定关系层：小范围 Typed Graph
长期知识层：Versioned Compiled Wiki
```

### 5. 错误代价有多高

个人研究笔记可以接受少量编译偏差；自动执行生产回滚不行。风险越高，越要加强：

- Source Span 级 Citation；
- 人工 Review；
- Revision Pinning；
- Permission / Approval；
- Dry Run 与 Evidence；
- 无证据拒绝执行。

### 6. 团队有没有维护中间表示的能力

Graph 和 Compiled Wiki 都不是“一次导入，永久准确”。它们需要 Owner、Schema、Lint、Eval、重建和迁移。

如果没人负责，最朴素的 Hybrid RAG 往往比一套陈旧的高级知识系统更可靠。

## 我会怎样设计生产架构

生产架构需要提供一个统一的 Knowledge Contract：

```ts
type KnowledgeQuery = {
  query: string;
  intent?: 'lookup' | 'relation' | 'procedure' | 'global';
  scope: {
    tenantId: string;
    workspaceId: string;
    revision: string;
    principalId: string;
  };
  budget: {
    maxTokens: number;
    maxLatencyMs: number;
  };
};

type Evidence = {
  claim?: string;
  content: string;
  sourceUri: string;
  sourceVersion: string;
  sourceSpan?: { start: number; end: number };
  retrievalPath: string[];
  confidence: number;
};
```

Gateway 再按 Intent 路由：

```text
lookup
  → Hybrid Search

relation / impact
  → Entity Resolve
  → Graph Traverse
  → Raw Evidence Join

procedure / repeated synthesis
  → Compiled Index
  → Read Page
  → Follow Links
  → Critical Claim Source Check

global
  → Community Summary / Compiled Overview
  → 必要时分解 Subquery 回查原文
```

返回值统一为 Evidence，而不是让某个后端直接生成最终答案。这样权限、版本、引用、预算和 Trace 才不会散落在各条路线里。

## 分阶段落地：不要第一天同时建三套

### Phase 0：先建评测集

从 30–50 个真实问题开始，标注必要 Evidence、允许版本、禁止答案和拒答条件。没有这一步，后续只能靠“看起来更聪明”做选型。

### Phase 1：Hybrid RAG

完成 Parse、Chunk、BM25、Vector、Metadata Filter、Rerank、Citation。先把单事实和局部解释做好。

升级信号：

- 必要证据经常分散在三跳以上；
- 影响分析靠调大 Top K 仍不稳定；
- 大量问题反复综合同一批资料。

### Phase 2：只为高价值关系建图

先选一个子域，例如服务依赖或法规引用。使用 Typed Edge，保留 Source Span，不追求覆盖所有名词。

验收条件：

- 影响路径比 Hybrid RAG 的 Recall 更稳定；
- Edge Precision 达到业务可接受标准；
- 增量更新耗时和成本可控。

### Phase 3：编译高频知识

从 10–20 个高频 Topic / Procedure 开始。每个编译产物必须有 Source Manifest、Compiler Version、Status 与 Eval。

不要一上来把全部文档自动改写成 Wiki。编译范围越大，信息损失和陈旧页面越难定位。

### Phase 4：让路由器学会组合

复杂问题可以先查编译页获得结构，再回 Vector Index 找最新原文，最后沿 Graph 验证影响路径。

```text
Compiled Page 提供计划
      ↓
Graph 提供关系范围
      ↓
Hybrid Search 提供当前证据
      ↓
Evidence Gateway 合并、去重、校验
```

到这个阶段，三条路线才形成互补。

## 常见误判

### “资料少，就用向量 RAG”

资料少但关系密集，例如几十个系统组件的依赖分析，Graph 仍可能有价值。资料量不是唯一变量。

### “有 Graph 就不用 Vector”

图谱负责结构，原文负责细节。微软 GraphRAG 的 Local Search 本身也会组合 Knowledge Graph 与原始 Text Unit。Graph 不能替代所有文本证据。

### “编译一次，以后查询更便宜”

不一定。一项 2026 年的小规模预注册比较发现，LLM-compiled Wiki 更擅长跨论文连接与 Claim 级引用对齐，但测试设置下每次查询使用的 Token 反而明显高于 RAG；分解式 RAG 又追回了大部分综合能力差距。研究的核心结论很克制：组织证据、引用支撑和成本是三项不同能力，没有一个架构全胜。详见 [Vector RAG vs LLM-Compiled Wiki](https://arxiv.org/abs/2605.18490)。

编译能否回本，要用自己的 Query 分布、更新频率和 Review 成本测。

### “Wiki 已经整理好了，可以直接执行”

Wiki 是派生知识，不是权限系统。Agent 在生产环境执行动作，仍要经过 Permission、Approval、Dry Run 和 Evidence 记录。

### “让 LLM 自动维护，维护成本接近零”

LLM 降低了写页面和补链接的成本，却增加了事实校验、冲突处理、Schema 演进和错误传播风险。自动化的是体力活，不是责任。

## 要不要用 / 我的判断框架

如果今天从零做一个知识助手，我不会把“向量、Graph、编译”写进同一张采购清单。

**值得马上做：**

- Hybrid Search；
- Metadata / Permission / Revision Filter；
- Source Span Citation；
- 一组包含拒答和旧版本干扰的 Eval。

**出现明确瓶颈再做：**

- 多跳依赖和影响分析不稳定时，加 Typed Graph；
- 高频综合和流程每次都被重复推理时，加 Versioned Compiled Knowledge；
- 跨整个语料的主题总结成为核心需求时，再评估 Community Report / Global Search。

**无论选哪条路线都不能省：**

- Raw Source 与派生索引分离；
- Evidence 能回到 Source；
- 索引和编译产物可重建；
- 查询绑定 Scope、Revision 与 Permission；
- 没有足够证据时明确拒答。

最后的架构判断可以压缩成一句话：

> Vector RAG 解决“相关内容在哪里”，GraphRAG 解决“这些对象怎么关联”，编译式 RAG 解决“哪些稳定知识值得提前整理并持续复用”。

先让真实问题暴露知识形状，再选择表示方式。技术路线不应该从一张产品能力表里长出来，而应该从失败用例和评测数据里长出来。
