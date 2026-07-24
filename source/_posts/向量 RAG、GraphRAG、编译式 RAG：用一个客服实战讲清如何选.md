---
title: 向量 RAG、GraphRAG、编译式 RAG：用一个客服实战讲清如何选
date: 2026-07-24 14:00:00
description: 向量 RAG、GraphRAG 和编译式 RAG 并不是一条从低级到高级的升级路线。本文用一个脱敏的跨行业客服知识助手贯穿三种方案，拆解知识问答、订单关联和服务流程如何命中证据，并给出从 MVP 到生产架构的选型矩阵、评测方法与组合策略。
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

这篇文章不再横向罗列名词。我会用同一个“跨行业客服知识助手”贯穿三条路线，观察它们如何处理同一批文档、回答三类问题，再给出我会如何从 MVP 演进到生产架构。

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

## 读实战前，先认清这些专业术语

RAG 相关术语容易混在一起，是因为它们来自不同层：有些描述数据，有些描述检索算法，有些描述知识结构，还有一些负责权限和证据治理。先把它们放回一条完整链路：

```text
Raw Source
  → Parse / Chunk
  → Index
  → Query
  → Retrieve / Rank
  → Evidence Context
  → LLM Generate 或 Agent Action
```

这条链路里，Source 是事实，Index 是为了查询而生成的派生数据，Evidence 是本次回答实际采用的依据。三者不能混为一层。

### RAG、Context 与 Agent

| 术语 | 通俗解释 | 在客服实战中的位置 |
| --- | --- | --- |
| RAG | Retrieval-Augmented Generation，先检索外部资料，再让 LLM 基于资料生成答案 | 回答退款政策前，先取回有效政策片段 |
| Retrieval | 根据 Query 从外部知识中找候选内容 | 找到退款时效、订单状态和活动例外 |
| Generation | LLM 根据问题与 Evidence 组织自然语言答案 | 把政策改写成客服能直接使用的回复 |
| Context Window | 模型单次推理能接收的 Token 范围 | 决定一次能放入多少问题、对话和证据 |
| Context Budget | 系统主动分配给证据的上下文额度 | 在有限 Token 中优先放最关键的政策和例外 |
| LLM | Large Language Model，负责理解与生成，不自动等于事实源 | 解释证据、生成回复，但不能凭空创造政策 |
| Agent | 能调用 Tool、读取结果并继续决策的 LLM 运行循环 | 查询知识后，再读取订单状态或创建升级建议 |
| Copilot | 嵌在现有工作台中辅助人的 Agent 形态 | 在当前工单里推荐字段、证据和下一步动作 |
| Tool | Agent 可调用的受控接口 | 只读查询订单状态，或在审批后执行某项业务动作 |

RAG 只规定“检索后再生成”这条工作流，并没有规定一定使用向量数据库。BM25、Vector Search、Graph Search，甚至直接读取编译好的 Wiki，都可以成为 Retrieval 的实现。

### 从文档到索引

| 术语 | 通俗解释 | 容易踩的坑 |
| --- | --- | --- |
| Raw Source | 未被检索系统改写的原始事实源 | 把自动摘要当成原始政策，导致错误无法追溯 |
| Parse | 从 Markdown、PDF、HTML、表格等格式中提取正文与结构 | 表格行列、标题层级和脚注在解析时丢失 |
| Chunk | 为检索切出的内容单元，不一定等于固定字数 | 切得太小会失去条件，太大会混入无关内容 |
| Metadata | 附在内容上的结构化信息，如类型、时间、权限和版本 | 只算相似度，不过滤过期或无权限政策 |
| Index | 为搜索构建的派生结构，如倒排索引、向量索引或图索引 | 把 Index 当事实源；正确做法是允许删除后重建 |
| Ingestion | 从 Source 到 Parse、Chunk、Metadata 和 Index 的整条构建流水线 | 只做首次导入，没有增量更新和失败重试 |

以退款政策为例，一个 Chunk 不应只剩“预计 1–3 个工作日到账”。它还要带上“原路退款”“演示会员等级”“支付渠道结果为准”等条件，并保留 Source Version。否则检索命中了数字，回答仍可能是错的。

### 检索与排序

| 术语 | 通俗解释 | 它擅长什么 |
| --- | --- | --- |
| Query | 用户问题或系统改写后的搜索表达式 | 表达这次到底要找什么 |
| Query Rewrite | 把口语问题改写成更适合搜索的一组表达式 | 补充别名、标准术语或拆出多个子问题 |
| Keyword Search | 按出现的词查找内容 | 状态码、产品编码、政策名称等精确词 |
| BM25 | 根据词频、词的稀有程度和文档长度计算关键词相关性 | 把包含关键术语的文档排到前面 |
| Embedding | 把文字映射成一组浮点数，用距离近似语义相似度 | 连接“退款多久到”和“退款到账时效”等不同说法 |
| Vector Search | 比较 Query Vector 与文档 Vector，找语义接近的内容 | 自然语言问法、同义表达和模糊描述 |
| Vector Database | 存储向量并执行近邻搜索的数据库或索引能力 | 解决向量的存取与查询，不负责答案正确性 |
| Metadata Filter | 在召回前后按租户、时间、权限、版本等条件过滤 | 阻止测试规则、旧政策或越权内容进入上下文 |
| Hybrid Search | 同时使用 Keyword / BM25 与 Vector Search | 兼顾精确术语和自然语言语义 |
| Rank Fusion | 把多个检索器的排名合并成一个候选列表 | 合并 BM25 与 Vector 的结果，而不是简单拼接 |
| Reranker | 用更精细的模型对少量候选重新判断相关性 | 把“语义相近”进一步收窄为“能回答当前问题” |
| Top K | 最终保留排名最靠前的 K 条结果 | 控制 Recall、噪声和 Context Budget 的平衡 |

Embedding 不会“理解并保存答案”。它只提供一种相似度信号。Reranker 也不能找回第一轮完全没召回的资料。因此检索评测要分开看 Recall 和排序质量。

### 图、编译与知识表示

| 术语 | 通俗解释 | 在本文中的例子 |
| --- | --- | --- |
| Entity / Node | 可以独立识别的业务对象 | 订单、会员、权益券、政策 |
| Relationship / Edge | 两个 Entity 之间有类型和方向的关系 | 订单使用权益券、权益券受活动政策约束 |
| Knowledge Graph | 由 Entity、Relationship 及其属性组成的知识网络 | 表达订单、商品、会员、权益与政策的连接 |
| Multi-hop | 为回答问题连续经过多条关系 | 取消订单 → 活动权益券 → 活动政策 → 退回条件 |
| GraphRAG | 用图结构定位关系，再结合原文 Evidence 生成答案 | 解释某类权益为什么没有随订单取消而退回 |
| Community | 图中连接紧密的一组实体 | 围绕退款、支付或会员权益形成的对象集合 |
| Community Report | 对某个 Community 预先生成的主题摘要 | 用于回答“这批客服资料有哪些主要问题” |
| Knowledge Compilation | 在构建阶段把 Raw Sources 整理成稳定的知识产物 | 把多份政策编成可审阅的支付异常处理 Procedure |
| Intermediate Representation | Source 与最终消费之间的中间表示，简称 IR | Topic Page、Entity Page、Procedure、Conflict Page |
| Build Artifact | 一次编译输出的、可版本化和可重建的产物 | Wiki、Knowledge Pack、索引和 Citation Map |
| LLM Wiki | 由 LLM 增量维护、相互链接的结构化 Wiki | 让重复查询优先复用已有页面，再回原文核验 |

GraphRAG 和编译式 RAG 都会在 Query 发生前做更多加工。区别是 GraphRAG 主要预构建关系，编译式 RAG 还会预先合并事实、流程、冲突和解释结构。

### Evidence、Citation 与治理

| 术语 | 通俗解释 | 为什么需要它 |
| --- | --- | --- |
| Evidence | 本次回答实际使用、能够支撑某个 Claim 的内容 | 判断答案是不是基于资料，而不是模型猜测 |
| Claim | 回答里可以被验证真假的具体陈述 | “退款预计 1–3 个工作日到账”就是一个 Claim |
| Citation | 从 Claim / Evidence 回到 Source 的可定位引用 | 让客服和审核人检查原文、章节与版本 |
| Grounding | 回答中的 Claim 是否被给定 Evidence 支撑 | 区分“答得像真的”和“确实有依据” |
| Scope | 一次查询被允许使用的业务、租户或知识范围 | 防止不同业务空间的规则互相污染 |
| Revision | 本次查询固定使用的知识版本 | 避免新旧政策混在同一个答案里 |
| Permission | 当前主体是否有权读取或执行 | 有权看通用政策，不代表有权看真实订单 |
| Approval | 高风险动作执行前的明确批准 | 退款、补偿和改订单不能由知识命中自动触发 |
| Trace | 记录 Query、路由、候选、Evidence 和最终输出的链路 | 出错后能定位是召回、排序、编译还是生成问题 |
| Knowledge Gateway | 统一接收知识查询并治理多个后端的入口 | 屏蔽 Hybrid、Graph 和 Compiled Wiki 的接口差异 |

Evidence 不等于“搜索结果”，Citation 也不只是文章末尾列几个链接。理想状态是回答中的关键 Claim 能逐条对应具体 Evidence，Evidence 再指向带 Revision 的 Source Span。

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
| Source Code | 产品说明、服务政策、FAQ、流程规范和已审核案例等原始事实源 |
| Intermediate Representation | Topic Page、Entity Page、Procedure、Decision、Conflict、Citation Map |
| Compiler | LLM + 确定性解析器 + Schema + 校验器 |
| Build Artifact | 可被人阅读、被 Agent 搜索和遍历的 Wiki / Knowledge Pack |

它不是“预先做了 Embedding”的新名字。Embedding 只是建立向量索引；编译式 RAG 会合并跨文档事实、建立链接、显式记录冲突，甚至把陈述性知识整理成可执行流程。

它也不能自动取代 RAG。编译过程会丢信息、写错关系、把条件句压成绝对规则。2026 年的 [WiCER 研究](https://arxiv.org/abs/2605.07068)专门讨论了这个 compilation gap：盲目压缩原始知识会出现严重信息损失，必须用诊断问题找出丢失事实，再迭代修正编译产物。

## 实战场景：给客服做一个知识助手

假设我们要做一个跨行业客服知识助手。它不绑定某家公司的产品、渠道或组织，示例中的会员等级、订单、规则、数值和 URI 均为虚构，仅用于说明架构。

团队准备了八份经过脱敏的资料：

| ID | 文档 | 主要内容 |
| --- | --- | --- |
| D1 | `product-catalog.md` | 虚构产品、服务权益与适用范围 |
| D2 | `refund-policy.md` | 退款方式、预计时效与解释口径 |
| D3 | `membership-policy.md` | 虚构会员等级与权益限制 |
| D4 | `order-status.md` | 订单状态、支付状态与状态转换 |
| D5 | `resolved-cases.md` | 已审核、已脱敏的历史客服案例 |
| D6 | `business-relations.yaml` | 订单、商品、会员、权益券和规则的关系 |
| D7 | `service-sop.md` | 核验、回复、升级与留痕步骤 |
| D8 | `campaign-exceptions.md` | 虚构活动订单的例外和覆盖规则 |

为了避免把 Demo 做成“上传 PDF，然后问一句摘要”，我选三类形状完全不同的问题：

```text
Q1 事实查找：
虚构的 Demo Plus 会员申请原路退款后，预计多久能到账？

Q2 关系与影响：
订单取消后，为什么某类活动权益券没有退回？要看哪些规则？

Q3 稳定执行：
用户已付款，但订单仍显示“待支付”，客服应该如何核验、回复和升级？
```

这三问分别要求系统找到一个事实、走一条关系链、复用一套跨文档流程。它们比“总结这些文档”更接近真实系统。

这套实战也刻意划出产品边界。客服场景不能被粗暴地收敛成一个聊天框：

| 能力 | 合适的入口 | 知识系统负责什么 |
| --- | --- | --- |
| 通用知识问答与推荐 | 独立知识问答入口 | 查询产品说明、政策与标准口径 |
| 当前工单的信息提取 | 工单内 Copilot | 读取已脱敏上下文，推荐字段和 Evidence |
| 订单、会员与权益关联解释 | 工单内 Copilot + 受控业务 Tool | Graph 解释关系，Tool 提供当前状态 |
| 退款、补偿、改订单等动作 | 有权限和审批的操作流 | 知识只提供规则依据，不直接取得执行权 |

时效要求不高、跨行业共用的知识问答可以沉到通用平台。和当前会话、订单状态、退款流程强绑定的能力，应留在客服工作台 Copilot 中。RAG 提供依据，业务 Tool 读取实时状态，Permission / Approval 决定能否执行。

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

生产环境我不会只留向量通道。订单状态码、产品编码、政策编号和标准术语更适合 BM25，因此实际链路会是：

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
  "chunkId": "refund-policy#original-channel",
  "content": "演示规则：Demo Plus 会员的原路退款预计 1–3 个工作日到账；实际时间以支付渠道处理结果为准。",
  "sourceUri": "demo://policies/refund",
  "sourceVersion": "sha256:...",
  "section": "原路退款时效",
  "scope": ["customer-service-demo"],
  "validFrom": "2026-06-01",
  "permission": "engineering"
}
```

没有这些 Metadata，检索器可能把旧版本、测试环境或无权限文档排在最前面。相似度高，不等于业务上可用。

### 三个问题会发生什么

Q1 很适合 Vector / Hybrid RAG。Query 中的“原路退款”“多久能到账”和 D2 的表述足够接近，Top K 很容易命中正确段落。若客服搜索某个标准状态码，BM25 通道还能兜住精确标识符。

Q2 开始吃力。向量检索可能召回 `change-184.md` 和 `service-topology.md`，但它不会天然执行这条路径：

```text
取消订单
  → 使用了活动权益券
  → 权益券受活动规则约束
  → 活动规则规定部分场景不自动退回
```

Top K 里只要漏掉一段，影响分析就断了。把 `k` 调大能提高 Recall，却会带来更多无关上下文和更高推理成本。

Q3 的问题更明显。支付与订单状态定义在 D4，核验和升级动作在 D7，解释口径在 D2，活动例外又可能覆盖默认规则。单轮 RAG 每次都要重新找到多份资料并现场拼装流程。

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
(CancelledOrder)
  -[USED]-> (CampaignVoucher)

(CampaignVoucher)
  -[GOVERNED_BY]-> (CampaignPolicy)

(PlusMember)
  -[ELIGIBLE_FOR]-> (RefundBenefit)
```

查询 Q2 时，系统先定位“已取消订单”和“活动权益券”，再按允许的 Edge Type 遍历，最后把路径关联的原文片段一起交给模型。此时“为什么没有退回”不再靠模型从若干相似 Chunk 中猜，而是有一条显式路径。

微软的 [GraphRAG 官方索引流程](https://microsoft.github.io/graphrag/index/overview/)会从原文抽取 Entity、Relationship 和 Claim，做 Community Detection、生成 Community Report，并保留向量表示。它的 [Local Search](https://microsoft.github.io/graphrag/query/local_search/)会结合实体关系与原始 Text Unit，适合具体实体问题；Global Search 则基于 Community Report 做 Map-Reduce，更适合“整批资料有哪些主要主题”这类全局问题。

### GraphRAG 不是“Vector RAG 加一个图数据库”

难点不在于把 Node 和 Edge 存进去，而在于：

1. **实体消歧**：“权益券”“优惠权益”和“活动凭证”是不是同一类对象？
2. **关系方向**：A 调用 B 与 B 依赖 A 不能写反。
3. **时间有效性**：v2 的关系不能污染 v3。
4. **来源追踪**：每条 Edge 必须能回到产生它的 Source Span。
5. **增量更新**：一条政策发布新版本后，哪些社区摘要和关系要重算？
6. **权限裁剪**：删掉无权访问的 Node 后，剩余路径是否还泄露信息？

如果团队不能维护这些约束，GraphRAG 会把“检索偶尔漏片段”变成“图谱稳定地提供错误关系”。

### 我会怎样限制第一版图谱

不从所有自然语言里自由抽取任意关系，而是先定义少量 Typed Edge：

```ts
type EdgeType =
  | 'BELONGS_TO'
  | 'PURCHASES'
  | 'USES'
  | 'GOVERNED_BY'
  | 'ELIGIBLE_FOR'
  | 'SUPERSEDES'
  | 'ESCALATES_TO';
```

结构化的商品、订单状态和政策引用用确定性解析器产生；只有已脱敏客服案例、FAQ 等非结构化资料才交给 LLM 抽取，并标记较低 Confidence，等待审核。

这样做牺牲了“自动构建万物图谱”的想象力，换来可验证的影响分析。

## 第三轮：编译式 RAG 改变了什么

Q3 是编译式 RAG 最有价值的地方。团队已经反复处理“支付成功但订单状态未同步”，系统没必要每次从多份原文重新拼出核验与升级流程。

构建阶段可以产出：

```text
wiki/
  products/
    example-product.md
  policies/
    refund-policy.md
    membership-policy.md
  procedures/
    payment-order-mismatch.md
  decisions/
    policy-revision.md
  cases/
    reviewed-payment-case.md
  conflicts/
    campaign-exceptions.md
  index.md
  log.md
```

其中 `procedures/payment-order-mismatch.md` 不是一段随意摘要，而是带来源和适用条件的结构化产物：

```yaml
---
type: procedure
status: reviewed
revision: 7
applies_to:
  scenario: payment_succeeded_order_pending
  excluded_cases:
    - offline_payment_demo
source_refs:
  - demo://orders/status#pending
  - demo://policies/refund#original-channel
  - demo://sop/payment-check#escalation
compiled_from:
  - sha256:...
  - sha256:...
---
```

正文再写清：

```text
触发条件
1. 用户提供的脱敏支付凭证显示已付款；
2. 订单在演示用正常同步窗口后仍为“待支付”；
3. 当前订单不属于已声明的例外支付方式。

核验与回复
1. 通过只读工具核对订单与支付状态；
2. 不要求用户提供完整卡号、证件号等敏感信息；
3. 告知当前状态、预计处理路径和下一次反馈时间；
4. 状态持续不一致时升级到对应处理队列。

验证与留痕
1. 订单与支付状态最终一致；
2. 工单关联本次查询所用 Evidence；
3. 面向用户的回复不暴露内部系统和规则标识；
4. 敏感字段在日志与摘要中保持掩码。
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
| Q1 退款预计多久 | D2 的一个段落 | 最自然 | 可以，但过度设计 | 可以，仍应核验原文 |
| Q2 权益券为什么没退 | D4 + D6 + D8 的规则路径 | 容易漏一跳 | 最自然 | 若已有 Policy Pages 也能做 |
| Q3 如何核验、回复和升级 | D2 + D4 + D7 + D8 | 每次重新拼装 | 图能找关联，但流程顺序仍需组织 | 最自然，Procedure 可审阅复用 |

还要加入反向问题：

```text
Q4：订单刚显示“待支付”一分钟，是否应该立即承诺退款？
```

正确行为不是从“状态不一致”直接跳到退款，而是先核对演示规则中的正常同步窗口和支付状态，并明确不能在证据不足时承诺处理结果。

再加入冲突问题：

```text
Q5：活动订单取消后，权益券是否一定自动退回？
```

系统必须同时找到默认权益规则和 D8 的例外规则，按有效 Revision 判断谁覆盖谁。如果它只返回默认政策，即使语言很流畅也算失败。

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

一个系统最终答对了，不代表索引正确。LLM 可能靠训练知识猜中答案。Eval 数据里要包含专门构造、无法靠常识猜中的值，加入旧版本干扰项和“应该拒答”的问题，才能测出 Grounding。

建议把用例写成可执行契约：

```json
{
  "id": "support-payment-order-mismatch",
  "query": "用户已付款但订单仍显示待支付，客服应该如何处理？",
  "scope": {
    "domain": "customer-service-demo",
    "revision": "policy-demo-r7"
  },
  "expectedEvidence": [
    "order-status#pending",
    "refund-policy#original-channel",
    "service-sop#payment-check",
    "campaign-exceptions#payment-method"
  ],
  "mustMention": [
    "先核对订单与支付状态",
    "告知下一次反馈时间",
    "持续不一致时升级"
  ],
  "mustNotDo": [
    "索取完整支付卡号",
    "在证据不足时承诺退款"
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

商品、会员、订单、权益与正式政策之间的关系相对稳定，值得建图。客服对话里临时提到的“可能相关”，通常不值得成为长期 Edge。

关系变化比查询还快时，GraphRAG 的增量维护成本会吞掉收益。

### 3. Query 是否重复

编译的收益依赖复用。同一批资料只问一次，直接 RAG 通常更便宜；团队每周都要重新综合同一套规则，提前编译才开始划算。

这里不能只算 Token。还要算人工复核、错误传播和旧知识清理成本。

### 4. Source 是否持续变化

变化快的订单状态、库存和工单进度，应该通过受权限控制的实时 Tool 查询。稳定的产品说明、正式政策、领域概念和高频 Procedure，更适合编译。

一个常见组合是：

```text
快速变化层：实时 Tool 查询当前业务状态
稳定关系层：小范围 Typed Graph
长期知识层：Versioned Compiled Wiki
```

### 5. 错误代价有多高

个人研究笔记可以接受少量编译偏差；自动退款、赔付或修改订单不行。风险越高，越要加强：

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

先选一个子域，例如订单—权益—政策关系。使用 Typed Edge，保留 Source Span，不追求覆盖所有名词。

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

资料少但关系密集，例如商品、会员、订单、活动与售后政策之间的关联查询，Graph 仍可能有价值。资料量不是唯一变量。

### “有 Graph 就不用 Vector”

图谱负责结构，原文负责细节。微软 GraphRAG 的 Local Search 本身也会组合 Knowledge Graph 与原始 Text Unit。Graph 不能替代所有文本证据。

### “编译一次，以后查询更便宜”

不一定。一项 2026 年的小规模预注册比较发现，LLM-compiled Wiki 更擅长跨论文连接与 Claim 级引用对齐，但测试设置下每次查询使用的 Token 反而明显高于 RAG；分解式 RAG 又追回了大部分综合能力差距。研究的核心结论很克制：组织证据、引用支撑和成本是三项不同能力，没有一个架构全胜。详见 [Vector RAG vs LLM-Compiled Wiki](https://arxiv.org/abs/2605.18490)。

编译能否回本，要用自己的 Query 分布、更新频率和 Review 成本测。

### “Wiki 已经整理好了，可以直接执行”

Wiki 是派生知识，不是权限系统。Agent 查询真实订单或执行退款、赔付等动作，仍要经过 Permission、Approval、Dry Run 和 Evidence 记录。

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
