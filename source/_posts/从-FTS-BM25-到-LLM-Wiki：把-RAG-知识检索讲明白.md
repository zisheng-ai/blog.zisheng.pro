---
title: 从 FTS、BM25 到 LLM Wiki：把 RAG 知识检索讲明白
date: 2026-07-22 23:20:00
description: RAG、FTS、BM25、Vector Search、GraphRAG 和 LLM Wiki 经常一起出现，却解决着不同层次的问题。本文用一条从文档到答案的链路，把这些概念放进同一张地图，并结合 Autoship 说明什么时候该用关键词检索、语义检索、图谱，什么时候需要 Evidence 和 Revision。
categories:
  - [AI]
tags:
  - RAG
  - GraphRAG
  - LLM Wiki
  - FTS
  - BM25
  - Vector Search
  - AI Infra
cover: /images/rag-llm-wiki知识检索地图.webp
---

我最近在 Autoship 里接入 GBrain，连续遇到几个看起来很像、实际分工完全不同的词：RAG、FTS、BM25、Vector Search、GraphRAG、LLM Wiki、Evidence。

一开始我也容易把它们理解成一条“越来越高级的产品线”：RAG 是基础，GraphRAG 是升级，LLM Wiki 是终局。这个理解不准确。它们分别描述的是检索方式、数据结构、工程系统和知识治理方法，不能简单排成一条替代关系。

## 一句话总结

RAG 是“让 LLM 回答前先查资料”的工作流；FTS 和 BM25 是关键词检索与排序；Vector Search 是语义检索；GraphRAG 是利用实体和关系回答关联问题；LLM Wiki 是围绕事实源、Evidence、Revision 和引用建立的一套知识工程系统。

换句话说：

```text
事实源 → 索引/图谱 → 检索 → Evidence → LLM 回答 → Citation
```

其中，FTS、BM25、Vector Search、Graph Search 都可以是“检索”这一层的实现。LLM Wiki 负责把这条链路治理起来。

## 先从 RAG 说起

RAG 是 Retrieval-Augmented Generation，中文通常叫“检索增强生成”。它不是某个数据库，也不是某个 UI 产品，而是一种回答流程：

```text
用户问题
  ↓
检索相关资料
  ↓
把资料放进 Prompt
  ↓
LLM 基于资料生成答案
```

普通 LLM 只能依赖训练数据和当前对话上下文。RAG 让它在回答之前访问外部知识，因此适合项目代码、团队规范、内部文档、实时数据等场景。

RAG 最重要的不是“把文档塞给模型”，而是检索到正确的上下文。检索错了，LLM 可能会非常自信地基于错误资料回答；检索不到，模型又会回到自己的先验知识，产生幻觉。

所以 RAG 的核心指标通常不是模型有多大，而是：

- Recall：应该找到的资料有没有找到？
- Precision：找到的资料是不是确实相关？
- Groundedness：回答是否真的被检索结果支撑？
- Citation：用户能不能回到原始证据？

## FTS：全文搜索是什么

FTS 是 Full-Text Search，也就是全文搜索。

普通字符串搜索更像 `grep`：在文件里找一段文本。FTS 会先把文档分词、建立索引，再快速查找包含查询词的文档或 chunk。

例如查询：

```text
Projection Overlay Workspace Revision
```

FTS 会寻找包含这些词的文档片段。它特别适合搜索：

- 类名、函数名、变量名
- 错误码和配置项
- 技术术语
- 产品名和领域词
- 精确的 API 名称

FTS 的特点是“看词”，不真正理解语义。因此“Revision 是什么”和“Revision 的定义”未必会得到同样好的结果，中文自然语言问题也可能因为分词和语言配置而召回不稳定。

## BM25：给搜索结果排序

FTS 负责找候选结果，BM25 负责判断谁更相关。

BM25 是 Best Matching 25，一种经典的关键词相关性排序算法。它会综合考虑：

- 查询词是否出现在文档中
- 查询词出现了多少次
- 这个词是不是稀有词
- 文档长度是否过长

可以把它理解成：

```text
FTS：哪些文档包含这些词？
BM25：包含这些词的文档，谁最值得排在前面？
```

BM25 不理解“意思”，但它对代码和技术术语非常实用。搜索 `Workspace Revision` 时，BM25 往往比纯向量相似度更可靠，因为向量模型可能把一些语义相近但并不包含这个精确标识符的文本排得很前。

## Vector Search：按语义找内容

Vector Search 会用 Embedding Model 把问题和文档转换成向量，再比较它们在向量空间中的距离。

例如下面几个问题的关键词不完全相同，但语义接近：

```text
Revision 是什么？
一个 Workspace 的版本如何确定？
工作区如何固定一组可信资产？
```

纯 FTS 可能认为它们差异很大；Vector Search 有机会把它们召回到同一组文档。

但 Vector Search 也不是万能的。它可能忽略精确术语、混淆相似概念，还可能把“相关但不适用”的内容排到前面。因此工程上更常见的是 Hybrid Search：

```text
Hybrid Search = FTS/BM25 + Vector Search + Metadata Filter
```

Autoship 当前的本地 GBrain MVP 先使用 FTS，原因很务实：本地部署简单、依赖少、可解释，而且对项目里的英文领域术语已经有效。后续再加入 Embedding，解决中文自然语言和同义表达的召回问题。

## GraphRAG：回答关系问题

GraphRAG 不是“把所有 RAG 都换成图数据库”，而是把知识表达成实体和关系：

```text
Workspace Revision ──固定──> Asset Version
Overlay ──叠加于──> Revision
Projection ──生成自──> Revision
Evidence ──支撑──> Claim
```

当用户问的是“某个词是什么”时，FTS 或 Vector Search 通常已经够用；当用户问的是“几个对象之间有什么关系”“这条结论依赖哪些事实”“修改某个事实会影响什么”时，图结构就更有价值。

GraphRAG 的优势是关系可追踪、路径可解释，适合架构依赖、权限关系、组织知识、实体关联和影响分析。代价是数据建模和维护成本更高，图谱抽取也可能引入错误关系。

## LLM Wiki 到底是什么

LLM Wiki 更像一个知识工程系统，而不只是一个检索算法。它通常包含：

1. Fact Sources：事实源，例如代码仓库、语雀文档、钉钉文档、ADR、会议纪要。
2. Ingestion：把事实源同步、解析、切块和建立索引。
3. Retrieval：用 FTS、BM25、Vector Search 或 Graph Search 找资料。
4. Evidence：检索出来、可以支撑某个结论的证据片段。
5. Citation：证据的可回溯引用地址。
6. Revision：事实在某个时间点的版本和有效范围。
7. Governance：处理冲突、过期、权限和审核。

因此，LLM Wiki 的关键问题不是“模型能不能回答”，而是：

> 这条回答依据的事实是什么？事实来自哪里？什么时候有效？谁能修改？

这就是它和简单 RAG 的差异。简单 RAG 重点解决“找一些相关文本”；LLM Wiki 还要解决“哪些文本可信、如何引用、冲突怎么办、知识怎么演进”。

## Evidence 为什么重要

Evidence 可以翻译成“证据”，但在知识系统里它不是随便一段上下文，而是能够支撑某个 Claim 的可定位事实。

例如：

```text
Claim：Projection 不是真实事实源，而是可重建的派生视图。
Evidence：架构文档中的 Projection 定义段落。
Citation：gbrain://default/autoship-tauri2项目架构与实施方案
```

没有 Evidence，LLM 只是“说得像”；有 Evidence 和 Citation，回答才具备审计能力。

在 Autoship 里，我希望每一个关键回答都能回答三个问题：

- 依据是哪条 Evidence？
- 来源是哪一个事实源？
- 如果事实发生 Revision，当前回答是否仍然有效？

## 什么时候用什么

| 问题类型 | 优先技术 | 原因 |
| --- | --- | --- |
| 查函数名、错误码、配置项 | FTS + BM25 | 精确词匹配可靠、速度快 |
| 用自然语言找相似说明 | Vector Search | 能处理同义表达和语义相近问题 |
| 同时兼顾术语和语义 | Hybrid Search | 工程上最常见的综合方案 |
| 查询实体之间的关系 | GraphRAG | 关系和路径可解释 |
| 要求回答可引用、可审计、可演进 | LLM Wiki | 需要 Evidence、Citation、Revision 和治理 |
| Autoship 当前阶段 | FTS/BM25 + Evidence | 先解决本地、可解释、低依赖的知识接入 |

不建议一开始就上完整 GraphRAG。知识量还小、关系还没稳定时，先把事实源、分块、检索、引用和版本做好，收益更高。

## Autoship 的落地顺序

我现在给 Autoship 采用的顺序是：

```text
本地事实源
  ↓
GBrain 索引
  ↓
FTS/BM25 检索
  ↓
Evidence + Citation
  ↓
Rust Host 注入 Agent Turn
  ↓
Codex / Qoder 生成或执行
```

后续按实际问题升级：

```text
第一阶段：FTS/BM25，确保精确术语能找到
第二阶段：Embedding，提升中文和语义召回
第三阶段：Hybrid Search，结合关键词与向量
第四阶段：对稳定实体建立关系图
第五阶段：补齐 Revision、权限、冲突和审核
```

验证效果时，不要只看“回答听起来不错”。至少要测四件事：命中率、证据准确率、引用覆盖率、额外延迟。尤其要做反向问题：知识库没有答案时，系统能不能明确说不知道，而不是编造一个答案。

## 要不要用 / 我的判断框架

我的判断很简单：

- 只有精确技术词：先用 FTS + BM25。
- 用户问题表达多样：加入 Vector Search。
- 问题核心是实体关系：再加入 GraphRAG。
- 团队需要长期维护、追责和协作：建设 LLM Wiki。

LLM Wiki 不是 RAG 的“替代品”，而是把 RAG 变成可维护知识基础设施的一种方式。GraphRAG 也不是必选终点，它只是关系型问题的一种强力检索手段。

对 Autoship 来说，最重要的不是把所有名词都装进去，而是先让每次 Agent 执行都能拿到可信、可引用、与当前 Revision 对齐的 Evidence。知识系统的价值，最终体现在它能不能让 Agent 少猜一次、少改错一次、少重复问团队一次。
