---
title: AI Native 知识库：给人维护，给 Agent 调用，给双方追溯
date: 2026-07-23 08:32:44
description: 传统知识库围绕人阅读，AI 知识库增加问答，AI Native 知识库则把 Agent 作为一等消费者。本文梳理从 FTS、BM25、Vector Search、RAG、GraphRAG 到 Agentic Retrieval 的发展，给出 Source、Index、Evidence、Revision 与 Permission 的最佳实践，以及不同技术的选型和组合方法。
categories:
  - [AI]
tags:
  - AI Native
  - Knowledge Base
  - RAG
  - GraphRAG
  - Agentic Retrieval
  - Hybrid Search
  - AI Infra
cover: /images/ai-native-knowledge-base.webp
---

![AI Native 知识库参考架构：给人维护、给 Agent 调用、给双方追溯](/images/ai-native-knowledge-base-architecture.webp)

*AI Native 知识库参考架构：Source of Truth 与 Index 分离，Knowledge Gateway 统一处理 Query Rewrite、Scope、Permission、Context Budget、Evidence 与 Trace。*

最近几天，我围绕一个 Agent 应用连续讨论和验证了 FTS、BM25、Vector Search、RAG、GraphRAG、LLM Wiki、代码结构分析、Query Rewrite 与 Evidence。名词越查越多，最后暴露出的缺口却很具体：缺的并非更大的 Vector Database，而是一套能让 Agent 稳定取用知识、同时让人审查依据的基础设施。

从前端转向 Agent / AI Infra 后，我发现知识库的默认用户变了。人会打开目录、搜索标题、阅读页面；Agent 不会沿着侧边栏一页页翻文档，它需要一个带 Scope、版本、权限、证据和失败语义的接口。只做“把文档塞进向量库”，Demo 很快，系统却很难进入真实执行链路。

## 一句话总结

如果系统只是让 AI 搜文档、回答问题，叫 **AI 知识库** 足够准确；如果知识的生产、索引、检索、引用、版本、权限和反馈闭环都围绕 Agent 的运行方式设计，才应该叫 **AI Native 知识库**。

这套系统同时服务人和 Agent，只是职责不同：

```text
人维护事实源
    ↓
系统编译成 Agent 可检索的索引与工具接口
    ↓
Agent 返回带 Evidence / Citation 的回答或行动依据
    ↓
人和系统共同验证、纠错、发布新版本
```

> 给人维护，给 Agent 调用，给双方追溯。

## 到底叫 AI 知识库，还是 AI Native 知识库

“AI 知识库”是宽泛的产品分类，“AI Native 知识库”是更严格的架构判断。判断标准在于系统把谁当成一等消费者。

| 形态 | 主要消费者 | 典型能力 | 合适的名字 |
| --- | --- | --- | --- |
| Wiki / 文档站 | 人 | 目录、编辑、全文搜索、权限 | 知识库 |
| 文档问答 | 人通过 Chat UI 使用 | 文档上传、Vector Search、RAG 回答 | AI 知识库 |
| Agent 知识基础设施 | Agent + 人 | 多源检索、Scope、Revision、Evidence、Citation、Trace、Eval | AI Native 知识库 |

我会用六个条件判断一个系统是否真的 AI Native：

1. Agent 通过稳定的 Tool / API 获取知识，而不是抓取某个 Wiki 页面。
2. Source of Truth 与检索索引分离，索引损坏后可以重建。
3. 每次检索都绑定 Tenant、Workspace、Revision 和 Permission。
4. 返回的不只是答案，而是 Evidence、Citation、Source 与有效版本。
5. 系统能按问题做 Query Rewrite、拆分子问题、路由不同知识源。
6. 检索与回答进入 Eval、Trace 和反馈闭环，错误能够被定位和修复。

只满足“上传 PDF 后能聊天”，我不会把它叫 AI Native。这个词应该表达架构差异，而不是给普通 RAG 换一个更时髦的包装。

## 知识库究竟是给人看的，还是给 Agent 看的

我把职责拆成三层。

### 第一层：事实源给人维护

Spec、ADR、Repository、数据库记录、产品规则和正式流程，首先要能被 Owner 阅读、评审和修改。Markdown、Git、文档平台和业务系统依然重要，因为人需要理解完整上下文，也需要承担发布责任。

人需要在这一层回答五个问题：

- 谁写的；
- 谁批准的；
- 哪个版本有效；
- 冲突时听谁的；
- 原文在哪里。

如果原始内容只存在于 Vector Database 里，人无法审查，也无法稳定重建，这不是事实源，只是一份难以治理的派生数据。

### 第二层：索引和接口给 Agent 使用

Agent 不需要每次读取整本 Wiki。它需要结构化、可过滤、可预算的 Knowledge Context：

```json
{
  "query": "一次发布失败后应如何回滚？",
  "scope": {
    "workspace": "project-a",
    "revision": "rev-2026-07-23",
    "principal": "current-user"
  },
  "evidence": [
    {
      "content": "...",
      "sourceUri": "kb://runbook/release-rollback",
      "sourceVersion": "v7",
      "score": 0.91
    }
  ],
  "traceId": "ktrace_xxx"
}
```

这份结果要控制 Context Window 预算，也要允许程序检查。Agent 关心的是：哪些片段与当前任务相关、是否有权限、是否属于当前版本、证据能否支撑下一步行动。

### 第三层：Evidence 给双方验证

Evidence 是这两层之间的桥。对 Agent，它是生成回答和执行计划的 Grounding；对人，它是回到原始事实、判断答案是否可信的入口。

界面不能只展示一段自然语言答案，还应允许查看：

- 引用了哪些片段；
- 每个片段来自哪个 Source；
- 检索时绑定了哪个 Revision；
- 哪些 Query 被改写或拆分；
- 哪个检索器命中了它；
- Agent 最终使用了哪些 Evidence。

知识库的正文给人看，编译结果给 Agent 用，Evidence 则必须让双方都看得懂。

## 知识库技术发展历程

![知识库技术发展历程：从搜索系统到 Agentic Retrieval](/images/knowledge-stack-evolution.webp)

这段历史最容易被讲成一条“后浪淘汰前浪”的升级路线：Vector Search 淘汰 BM25，GraphRAG 淘汰 RAG，Agentic Retrieval 再淘汰 GraphRAG。这个理解不对。它们解决的是不同层次的问题，今天仍然会同时出现在一套系统里。

### 1990s–2010s：从倒排索引到 BM25

最早的问题很直接：文档很多，怎样快速找到包含某个词的内容？

Inverted Index 把“词 → 文档”建立映射，FTS 负责全文检索，BM25 负责根据词频、稀有程度和文档长度排序。它们不理解抽象语义，却非常擅长精确对象：函数名、错误码、配置项、订单号、API 名称和领域术语。

这类能力没有过时。只要系统里存在精确标识符，Keyword Search 就仍然是不可替代的召回通道。

### 2019–2020：Dense Retrieval 与 Vector Search

Embedding 把 Query 和文档映射到同一个向量空间，检索开始从“出现了同一个词”走向“表达了相近的意思”。2020 年的 [Dense Passage Retrieval](https://arxiv.org/abs/2004.04906) 让 Dense Retrieval 在开放域问答中成为重要路线。

Vector Search 解决了同义表达和自然语言问题。例如“怎么固定一次工作区使用的内容”和“Workspace Revision 是什么”可能指向同一概念，即使它们共享的关键词很少。

但语义相似不等于业务上正确。Vector Search 可能把相似概念混在一起，也可能让精确的类名和错误码沉到后面。它增加了一种召回能力，没有取代关键词检索。

### 2020–2022：RAG 把检索接到生成模型前面

2020 年的 [RAG 论文](https://arxiv.org/abs/2005.11401) 把 Parametric Memory 与可检索的 Non-parametric Memory 结合起来。工程上最朴素的 RAG 流程是：

```text
Query → Retrieve Top K → 拼进 Prompt → LLM Generate
```

从这里开始，知识库不只返回文档，还开始参与回答。系统质量也不再只由搜索相关性决定：错误的 Chunk 会诱导模型自信地答错，没有证据时模型是否拒答，Citation 是否真的支撑 Claim，都成为新的问题。

### 2023–2024：Production RAG

RAG 从 Demo 进入生产后，“选一个 Embedding Model + Vector Database”很快暴露出边界。生产链路开始加入：

- Keyword + Vector 的 Hybrid Search；
- RRF 或其他 Fusion；
- Cross-encoder / LLM Reranker；
- Metadata Filter；
- 结构化或 Contextual Chunking；
- Citation、Groundedness 与检索评测；
- 权限、版本、增量更新、延迟和成本控制。

[Elastic 的官方文档](https://www.elastic.co/docs/solutions/search/hybrid-search)直接把 Hybrid Search 定义为全文检索与 Vector Search 的组合，并推荐用 RRF 合并排名。[Anthropic 的 Contextual Retrieval](https://www.anthropic.com/engineering/contextual-retrieval)则把文档级上下文补回每个 Chunk，再同时建立 Contextual Embedding 和 Contextual BM25，并叠加 Rerank。

到这一步，RAG 已经从 Prompt 技巧扩展成 Search / Data / Eval 工程。

### 2024–：GraphRAG 处理关系和全局问题

GraphRAG 显式抽取 Entity、Relationship、Claim 和 Community，让检索能回答两类普通 Top K RAG 不擅长的问题：

- 局部关系：某个实体与哪些对象相关，依赖链是什么；
- 全局问题：整批资料有哪些主要主题、组织和冲突。

[Microsoft GraphRAG](https://microsoft.github.io/graphrag/)同时保留 Basic Search、Local Search、Global Search 和 DRIFT Search。Local Search 会把图中的实体关系与原始文本片段组合，Global Search 则基于 Community Report 做全局汇总。这本身就说明 GraphRAG 不是 RAG 的替代品，而是增加了适合关系型问题的索引与 Query Mode。

实体和关系抽取需要额外模型调用，图谱可能有噪声，增量更新、冲突和版本治理也更复杂。Microsoft 的官方入门文档直接提醒 GraphRAG 索引会消耗大量 LLM 资源。没有真实的关系查询，就不该为了架构图好看而上 GraphRAG。

### 2025–2026：Agentic Retrieval

普通 RAG 通常“搜一次”；Agentic Retrieval 会先理解任务，再决定搜什么、去哪里搜、是否继续追问。

```text
用户任务
  ↓
Query Planning / Rewrite
  ↓
拆分多个 Subquery
  ↓
选择共享知识、当前文件、代码结构或实时系统
  ↓
并行 Keyword / Vector / Hybrid Search
  ↓
Rerank + Evidence Merge
  ↓
返回 Citation 与 Activity Trace
```

[Azure AI Search 的 Agentic Retrieval](https://learn.microsoft.com/en-us/azure/search/agentic-retrieval-overview)已经把这个过程产品化：根据 Query 和 Conversation History 生成聚焦的 Subquery，并行执行 Keyword、Vector 或 Hybrid Search，经过 Semantic Rerank 后保留引用与执行日志。

简单查询直接走低成本路径更稳。多意图、多来源或需要多跳证据时，再让 LLM 参与检索规划。

## 这些技术之间到底是什么关系

我现在按层级理解它们，不再把这些名词排成版本号：

| 概念 | 所在层 | 它解决什么 | 它不负责什么 |
| --- | --- | --- | --- |
| FTS / BM25 | Retrieval | 精确词召回与排序 | 不理解深层语义 |
| Embedding / Vector Search | Retrieval | 同义表达与语义召回 | 不保证精确标识符优先 |
| Hybrid / RRF | Retrieval | 合并关键词与语义结果 | 不验证证据是否正确 |
| Reranker | Ranking | 对候选结果做更精细排序 | 不能召回候选集中不存在的内容 |
| RAG | Generation Workflow | 用检索结果约束生成 | 不等于知识治理 |
| GraphRAG | Structured Retrieval | 关系、多跳、全局主题 | 不适合默认替代所有检索 |
| LLM Wiki | Knowledge Engineering | 组织事实源、知识页面、引用和演进 | 不是一种单独的检索算法 |
| Agentic Retrieval | Runtime Orchestration | 改写、拆问、选源、迭代和留 Trace | 不应绕过权限与治理 |
| AI Native 知识库 | System Architecture | 把以上能力变成 Agent 可依赖的基础设施 | 不是某个数据库或框架名 |

“LLM Wiki”并不是像 BM25 那样有严格定义的算法名。它更像一种产品和知识工程形态：让 LLM 帮助摄取、整理、链接、总结和维护知识页面。它可以使用 RAG，也可以使用 GraphRAG，但不会让底层检索失去用武之地。

## 2026 年搭建知识库的最佳实践

### 1. 先定义事实源，再选 Vector Database

先列出哪些系统有资格声明事实：正式 Spec、ADR、当前 Repository、业务数据库、审批记录、Runbook。会议记录、聊天记录和自动总结通常只能算 Raw Knowledge，不能自动覆盖正式结论。

我通常采用这样的优先级：

```text
已发布 Knowledge Asset
> 当前 Repository Revision
> 经确认的业务记录
> Raw Document / Meeting / Chat
> 模型生成的 Summary
```

“更新时间更晚”不等于“权威性更高”。

### 2. Source、Asset 与 Index 必须分开

这三个对象很容易被混成“已经导入知识库”：

- Source：原始事实在哪里；
- Knowledge Asset：经过清洗、归属、审核和发布的知识版本；
- Index：为了检索生成的 Chunk、Embedding、Posting List、Entity 与 Edge。

Index 应该是可重建的 Projection，不应该成为第二事实源。Embedding Model、Chunking Strategy 或 Schema 变化时，正确动作是从 Source / Asset 重建索引，而不是让旧向量永久背负历史包袱。

### 3. Chunk 不是按固定字数切完就结束

Chunk 至少要保留标题层级、表格语义、代码符号、列表边界和来源位置。不同内容应该用不同策略：

| 内容 | 优先切法 |
| --- | --- |
| Markdown / Spec | 按 Heading 与语义段落 |
| API 文档 | 按 Endpoint / Symbol |
| 代码 | 按函数、类、模块与调用关系 |
| 表格 | 保留表头与相关行 |
| 长 PDF | 结构解析后再按 Section |

不要迷信一个全局 Chunk Size。OpenAI 的 [Retrieval 文档](https://developers.openai.com/api/docs/guides/retrieval)允许自定义 Chunk Size 和 Overlap，也支持文件 Attribute Filter；这类参数应该由 Eval 驱动，而不是从教程里复制一个数字。

如果 Chunk 脱离父文档后不知道自己在说谁，可以给它增加简短的 Context Prefix，再建立 Keyword 和 Vector Index。

### 4. Hybrid Search 应该是团队知识的默认候选

对于只包含精确技术术语的小型知识库，BM25 足够简单可靠；一旦用户会用自然语言、中文或同义词提问，Keyword + Vector 通常更稳。

推荐链路是：

```text
Keyword Candidates ─┐
                    ├→ RRF / Weighted Fusion → Reranker → Top Evidence
Vector Candidates ──┘
```

先并行扩大 Recall，再用 Reranker 提升 Precision。不要直接拿两个分数相加，BM25 Score 与 Vector Similarity 往往不在同一标度上。

### 5. Scope 与 Permission 要在检索前生效

先检索全部内容、再在答案阶段隐藏结果，是危险设计。过滤至少要覆盖：

- tenant / organization；
- workspace / project；
- source；
- revision / effective time；
- principal / role；
- confidentiality；
- content status。

安全边界应尽量在 Candidate Retrieval 前收紧，避免跨项目内容进入模型上下文。对 Agent 来说，这既是数据权限问题，也是 Prompt Injection 与错误行动的隔离问题。

### 6. Query Rewrite 先做确定性版本

中文、缩写和领域词经常让 FTS 零召回。第一阶段不必立刻引入昂贵的 LLM Query Planner，可以先从受治理的 Glossary 生成确定性映射：

```text
“工作区版本是什么”
→ “Workspace Revision”
```

先查改写结果，零命中再回退原始 Query。这样行为可解释、可测试，也能快速验证问题到底在分词、Query 还是内容本身。

当复杂 Query 占比上升，再增加 Multi-query、Subquery Planning 和 Source Routing。只有任务复杂度值得支付规划成本时，才进入 Agentic Retrieval。

### 7. Rerank 之前先保证 Candidate Recall

Reranker 只能重排已有候选。正确文档没有进入 Top 50，后面的模型再强也救不回来。

因此评测要分层：

```text
Retrieval Eval：正确 Evidence 是否进入候选集？
Ranking Eval：正确 Evidence 是否排到足够靠前？
Generation Eval：回答是否被 Evidence 支撑？
Agent Eval：行动是否遵守 Scope、Permission 与事实？
```

把最终回答的好坏全部归因于 LLM，会让 Search 问题长期藏在 Prompt 里。

### 8. 每个关键 Claim 都应能回到 Evidence

Citation 不是装饰性的文末链接。一个可用的 Citation 至少包含：

```text
source_uri + source_version + location + content_hash
```

如果 Source 更新后 Citation 指向另一段内容，审计就失效了。对于会触发写文件、发消息、部署或审批的 Agent，关键决策必须保留 Knowledge Trace：用了哪些 Evidence、哪个版本、如何生成结论。

### 9. 明确“没有答案”也是正确结果

知识库缺少证据时，系统应该返回 `no_evidence`，而不是用模型常识补一段看似合理的答案。评测集中必须有反向样本：

- 知识库根本没有答案；
- 只有过期版本有答案；
- 当前用户无权访问答案；
- 两个正式 Source 互相冲突。

这几类问题比普通命中更能检验知识系统是否可信。

### 10. 用 Eval 决定升级，不用架构焦虑决定升级

我会维护一组 Query → Expected Evidence 的离线数据，并持续观察：

| 指标 | 回答的问题 |
| --- | --- |
| Recall@5 / Recall@20 | 正确证据有没有被找回 |
| MRR / nDCG | 正确证据排得是否足够靠前 |
| Zero-hit Rate | 是否频繁完全无结果 |
| Citation Precision | 引用是否真的支撑 Claim |
| Grounded Answer Rate | 答案是否只使用已给证据 |
| No-evidence Accuracy | 没答案时能否正确拒答 |
| Scope Leakage | 是否混入其他项目或无权限内容 |
| Freshness Lag | Source 更新多久后可被检索 |
| P50 / P95 Latency | 检索给 Agent Turn 增加多少延迟 |
| Cost per Retrieval | Rewrite、Embedding、Rerank 与 Planner 的成本 |

我的当前门槛是：先用确定性 Query Rewrite；中文评测出现 `Recall@5 < 85%` 或 `Zero-hit Rate > 5%` 时，再把 Multilingual Embedding + Hybrid Search 提到下一阶段。这两个数字服务于当前项目，不是行业标准。

## 每个技术如何决策

| 技术 | 什么时候用 | 什么时候先别用 | 主要代价 |
| --- | --- | --- | --- |
| `grep` / Raw File Search | 当前 Workspace、文件刚变化、精确字符串 | 团队共享知识与自然语言问答 | 能力简单，结果缺少治理 |
| FTS + BM25 | 错误码、类名、配置项、领域词 | Query 大量使用同义表达 | 分词与语言适配 |
| Vector Search | 自然语言、跨语言、同义表达 | 精确 ID 与小规模纯术语库 | Embedding 成本与语义误召回 |
| Hybrid Search | 团队文档、代码说明、中文问答 | 数据量极小且 BM25 已达标 | 两套索引与 Fusion 调参 |
| Reranker | 候选集 Recall 已好、Top K 排序不稳 | 正确证据根本没进候选集 | 额外模型延迟和成本 |
| Contextual Retrieval | Chunk 离开父文档后语义残缺 | 文档短且 Chunk 本身自包含 | 索引时生成上下文的成本 |
| GraphRAG | 依赖链、影响分析、组织关系、全局主题 | 主要是定义查询和精确搜索 | 图谱抽取、更新与噪声治理 |
| Code Graph | 跨文件 Symbol、调用关系、模块依赖 | 只是找文本或当前未保存文件 | 语言解析器与增量更新 |
| Agentic Retrieval | 多意图、多源、多跳、需要迭代检索 | 单一事实查询、高 QPS 低延迟路径 | Planner 成本、不可预测性与 Trace 复杂度 |

我用下面这个顺序做技术决策：

```text
问题能否靠精确词回答？
  ├─ 能 → BM25 / Raw File Search
  └─ 不能 → 是否主要是语义改写？
       ├─ 是 → Vector + Hybrid
       └─ 否 → 是否需要关系或全局理解？
            ├─ 是 → Code Graph / GraphRAG
            └─ 否 → 是否需要跨多个 Source 拆问？
                 ├─ 是 → Agentic Retrieval
                 └─ 否 → 先检查 Source、Chunk 与 Metadata
```

很多“需要更强模型”的问题，最后根因其实是 Source 不可信、Chunk 丢上下文、Scope 错了，或 Query 用错了检索器。

## 这些技术应该如何搭配

落地时，我把系统拆成一个事实面、多个检索投影和一个统一 Knowledge Gateway。

```text
┌──────────────────── Human-owned Sources ────────────────────┐
│ Spec / ADR / Repository / Runbook / Business Data / Raw Doc │
└───────────────────────────┬───────────────────────────────────┘
                            ↓
                 Normalize / Validate / Publish
                            ↓
┌──────────────────── Knowledge Assets ────────────────────────┐
│ owner · trust · version · effective time · ACL · source URI │
└───────────────────────────┬───────────────────────────────────┘
                            ↓ rebuildable projections
        ┌───────────────────┼────────────────────┐
        ↓                   ↓                    ↓
   BM25 / FTS          Vector Index       Graph / Code Index
        └───────────────────┼────────────────────┘
                            ↓
┌──────────────────── Knowledge Gateway ───────────────────────┐
│ Scope Filter → Rewrite / Plan → Route → Fusion → Rerank     │
│ → Evidence Package → Citation → Knowledge Trace             │
└───────────────────────────┬───────────────────────────────────┘
                            ↓
                    Agent Runtime / Tools
                            ↓
               Answer / Plan / Action / Delivery
                            ↓
                       Eval & Feedback
```

BM25、Vector Database、Graph Engine 和本地搜索都收口到 Gateway；如果直接暴露给每个 Agent Provider，权限、版本、Citation 和审计会散落在不同 Adapter 里。

Provider 应该看到一个稳定的领域接口，例如：

```ts
type KnowledgeSearchRequest = {
  query: string;
  intent?: "lookup" | "explain" | "impact" | "global";
  workspaceId: string;
  revision: string;
  principal: string;
  budget: {
    maxResults: number;
    maxTokens: number;
    maxLatencyMs: number;
  };
};
```

具体使用 BM25、Vector、Graph 还是多路并行，由 Knowledge Gateway 根据 Intent、数据新鲜度、预算和 Eval 结果决定。

## 三种可直接落地的组合

### 组合 A：个人 / 小项目

```text
Markdown / Repository
→ SQLite FTS5 或本地 BM25
→ Glossary Query Rewrite
→ Top 5 Evidence + 文件行号 Citation
→ Agent
```

这套组合完全在本地运行，依赖少，检索行为也容易解释。内容不多时，不要先搭分布式 Vector Database。

### 组合 B：团队知识库

```text
Git / 文档平台 / 业务系统
→ 规范化 Knowledge Asset
→ BM25 + Multilingual Embedding
→ Metadata Filter + RRF + Rerank
→ Citation + Revision + ACL
→ Chat / Agent 共用
```

对多数团队，这已经是一条够用的 Production RAG 基线。先把 Hybrid、治理和 Eval 做稳，再讨论图谱。

### 组合 C：应用侧集成

```text
共享长期知识：团队 Spec / ADR / Runbook
当前工作区：实时 Raw File Search
代码结构：Symbol / Dependency Graph
实时事实：业务 API / Database Tool
                ↓
         Unified Knowledge Gateway
                ↓
 Query Planning / Source Routing / Evidence Merge
                ↓
      Agent Action + Knowledge Trace + Eval
```

四类信息的时效和结构不同：

- 共享知识回答“团队已经确认了什么”；
- Raw File Search 回答“当前磁盘上刚刚发生了什么”；
- Code Graph 回答“代码对象怎样连接、改动影响哪里”；
- 业务 API 回答“此刻真实世界的状态是什么”。

把它们全部塞进同一个 Vector Index，会失去新鲜度、结构、权限和事实边界。

## 一条务实的实施路线

### Phase 0：先做 Eval Set

准备 30–100 条真实问题，每条标注 Expected Source、Expected Evidence、允许的 Revision 和拒答条件。没有这一步，后续所有“效果不错”都只是观感。

### Phase 1：Keyword + Evidence

先打通 Source、Chunk、FTS/BM25、Citation 和 `no_evidence`。对中文和领域词加入确定性 Query Rewrite。

我把这一阶段的验收写成五条：

- 正确 Source 能进入 Top K；
- Citation 能回到原文；
- Scope 不串；
- 没证据时拒答；
- 索引可以重建。

### Phase 2：Hybrid + Rerank

当同义表达和中文召回成为主要问题，加入 Multilingual Embedding，用 RRF 合并 Keyword / Vector 候选。只有 Candidate Recall 达标后再上 Reranker。

### Phase 3：按问题增加结构索引

代码影响分析用 Code Graph，实体关系和全局主题用 GraphRAG。不要创建一个包治百病的“统一大图谱”。

### Phase 4：Agentic Retrieval

当任务确实需要多个 Source、多轮补证和动态路由，再引入 Planner。给每次检索设置 Token、Latency 和 Cost Budget，并完整记录 Activity Trace。

## 最容易踩的坑

### 把 Vector Database 当知识库

Vector Database 只保存一种检索投影。它不知道哪份文档是正式结论，也不天然理解权限、版本和冲突。

### 把 `simple` 当中文分词

取消英文 Stemming 或 Stop Words，不等于获得中文 Tokenizer。中文检索应通过真实 Query Eval 判断是先做 Rewrite、Multilingual Embedding，还是增加中文 BM25 Side Index。

### 把 GraphRAG 当默认终局

图谱最擅长关系与全局问题。大多数“某个概念是什么”“某个错误怎么修”仍然更适合 BM25、Vector 或 Hybrid。

### 让 Agent 自由搜索却没有 Scope

Agentic Retrieval 增加了自主性，也放大了跨项目污染、越权和 Prompt Injection 风险。Planner 可以决定怎么搜，不能决定自己有权看什么。

### 只测最终回答

一段流畅答案可能引用错文档。必须分别评测 Retrieval、Ranking、Grounding 和 Action。

### 索引不可重建

如果团队不敢删掉 Index 重建，说明事实源、转换流程或版本记录并不可靠。可重建是知识基础设施成熟度的重要标志。

## 要不要用 / 我的判断框架

值得现在做：

- 把系统定位为“人维护事实、Agent 调用知识、双方追溯 Evidence”；
- 明确 Source of Truth、Knowledge Asset 与 Index 的边界；
- 从 FTS/BM25、Citation、Scope 和 Eval 起步；
- 团队知识默认评估 Hybrid Search；
- 把当前文件、代码结构、共享知识和实时业务事实分开接入；
- 让所有 Provider 通过统一 Knowledge Gateway 使用它们。

可以再等：

- 在没有关系型 Query 证据前建设完整 GraphRAG；
- 在单次 Query 已足够时启用 LLM Planner；
- 为了“AI Native”这个名字重写已有 Wiki；
- 在没有 Eval Set 时频繁更换 Embedding Model 和 Vector Database。

重点关注：

- Permission 和 Revision 必须先于检索结果进入模型；
- Citation 必须指向稳定的 Source Version；
- `no_evidence` 是正常结果，不是系统故障；
- Agent 的每个关键行动都要能回到 Knowledge Trace。

我的责任划分已经很明确：**人负责事实源，Agent 使用检索接口，Knowledge Gateway 管住 Scope、Revision 和 Permission。** Agent 给出答案或准备执行动作时，再把 Evidence、Citation 和 Trace 交回给人。

是否 AI Native，可以用一次错误行动来检验：系统能不能沿着 Trace 找到它用了哪条 Evidence、哪个版本、哪个检索分支，并据此修正 Source 或索引。只能看到一段回答、查不到依据的产品，即使加了 Chat Box，也仍然只是文档问答。

## 参考资料

- [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401)
- [Dense Passage Retrieval for Open-Domain Question Answering](https://arxiv.org/abs/2004.04906)
- [OpenAI Retrieval Guide](https://developers.openai.com/api/docs/guides/retrieval)
- [Elastic Hybrid Search](https://www.elastic.co/docs/solutions/search/hybrid-search)
- [Anthropic Contextual Retrieval](https://www.anthropic.com/engineering/contextual-retrieval)
- [Microsoft GraphRAG](https://microsoft.github.io/graphrag/)
- [Azure AI Search Agentic Retrieval](https://learn.microsoft.com/en-us/azure/search/agentic-retrieval-overview)
