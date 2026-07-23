---
title: Raw Knowledge 如何变成可信知识资产：以及中文 FTS 为什么不是改个配置
date: 2026-07-22 23:55:00
description: 接入 GBrain 后，我真正困惑的不是如何执行 import，而是原始仓库、语雀文档和会议记录何时才算可信知识，以及中文问句为什么明明包含术语却检索不到。本文结合一个 Agent 应用的实测，拆清 Raw Knowledge、GBrain Page、Knowledge Asset、中文 FTS 与 Embedding 的边界，并给出一条可落地的项目知识接入路线。
categories:
  - [AI]
tags:
  - GBrain
  - Raw Knowledge
  - Knowledge Asset
  - FTS
  - Embedding
  - Hybrid Search
  - AI Infra
cover: /images/raw-knowledge-to-asset-chinese-fts.webp
---

最近我把 GBrain 接到一个 Agent 应用的应用侧，能够检索本地文档、给 Codex 注入上下文，也能在回答中带上 `gbrain://` Citation。

链路跑通之后，我反而遇到了两个更本质的问题：一个新项目有代码仓库、语雀、钉钉、Issue 和会议记录，这些 Raw Knowledge 到底怎样才算“进入知识库”？另一方面，中文文档明明已经导入，为什么问一句“Revision 是什么”却可能一条也搜不到？

这两个问题表面上分别属于知识治理和搜索，底层却指向同一个误区：**派生索引不是事实源，语义检索也不是全文检索。**

## 一句话总结

Raw Knowledge 导入 GBrain 后，只是变成可检索的 Page、Chunk、Embedding 和 Edge；只有经过归属、规范化、校验和发布，才成为应用侧的 Knowledge Asset。中文检索则需要 Query Rewrite、Multilingual Embedding 与关键词检索协作，单纯把 PostgreSQL FTS 从 `english` 改成 `simple` 并不会获得中文分词能力。

完整链路应该是：

```text
Raw Knowledge
    ↓ 归属、清洗、规范化
Knowledge Source
    ↓ GBrain import / sync
Page → Chunk → FTS / Embedding → Edge
    ↓ 整理、校验、审核、发布
Knowledge Asset Version
    ↓
Workspace Revision
```

这里存在两条可以并行、但不能混为一谈的流水线：一条负责“先搜起来”，另一条负责“让知识变可信”。

## 被 GBrain 索引，不等于成为 Knowledge Asset

这是我这次最重要的认知修正。

GBrain 认识的核心对象是：

```text
Source → Page → Chunk → Embedding / FTS → Edge
```

应用侧认识的核心对象则是：

```text
Knowledge Draft
→ Knowledge Asset
→ Knowledge Asset Version
→ Asset Manifest
→ Workspace Revision
```

两边的概念不能直接画等号。

一篇语雀文档被同步进 GBrain 后，可以被搜索、切块、建立向量和关系，但它依然可能过期、互相冲突、没有明确 Owner，甚至只是一场会议里的临时观点。它是 Raw Knowledge，不是已经发布的 Knowledge Asset。

如果把“进入索引”直接理解成“成为资产”，GBrain 数据库就会悄悄变成第二事实源。之后谁也说不清：正式 Spec 和会议记录冲突时听谁的？索引中的旧 Chunk 是否仍然有效？一个已经删除的页面能不能继续影响 Agent？

所以我现在采用一个更严格的边界：

| 对象 | 是什么 | 能否作为正式依据 |
| --- | --- | --- |
| Raw Knowledge | 仓库、文档、Issue、会议、对话等原始内容 | 不能默认信任 |
| GBrain Page / Chunk | 从 Source 生成的可检索派生数据 | 只能回到 Source 验证 |
| Knowledge Draft | 从 Raw Knowledge 提炼出的待审核结论 | 尚未发布 |
| Knowledge Asset Version | 已完成校验和发布的不可变知识版本 | 可以作为正式依据 |
| Workspace Revision | 固定一组 Asset Version 的不可变快照 | 决定当前 Workspace 使用什么 |

## 新项目的知识库应该怎样组织

一个新项目不应该只有一个不分来源的“大知识库”。至少应该按来源和信任级别拆成三类 Source：

| Source | 内容 | 信任级别 |
| --- | --- | --- |
| `team-specs` | 正式 Spec、ADR、团队规范 | 高 |
| `{project}-repo` | 当前代码、README、仓库文档 | 中高 |
| `{project}-raw` | 语雀导出、会议记录、Issue、调研材料 | 中低 |

当不同 Source 冲突时，我采用下面的优先级：

```text
正式 Knowledge Asset
> 当前 Repository Revision
> 团队 Raw Knowledge
> 历史会议和对话
```

“更新得晚”不等于“权威性更高”。一条昨天的会议讨论，不能因为时间更近就覆盖上周已经评审通过的 ADR。

对于语雀和钉钉，正确的数据流也不是“连接以后全部自动变成知识”：

```text
语雀 / 钉钉
→ Connector 拉取
→ 保存 document_id、source_url、author、updated_at
→ 转换成规范 Markdown
→ 导入 project-raw Source
→ 提取候选结论生成 Draft
→ 人工或规则校验
→ 发布 Knowledge Asset Version
```

LLM 可以帮助提取 Draft，但不能因为“总结得像”就自动获得发布权。

## GBrain 实际怎样处理一份 Raw Knowledge

一份适合导入 GBrain 的最小 Markdown 可以是：

```md
---
title: Workspace Revision
type: concept
tags:
  - example-workspace
  - architecture
source_uri: https://example.com/original-document
owner: architecture-team
---

## 一句话定义

Workspace Revision 是面向 Workspace 发布的不可变资产集合快照。

## 约束

- 由 Asset Manifest 固定所有 Asset Version。
- 本地只能激活校验通过的版本。

<!-- timeline -->

## 2026-07-22

首次确认该定义。
```

GBrain 导入时会解析 Front Matter、生成稳定 Slug、计算 Content Hash、创建 Page、拆分 Chunk，然后建立 Keyword Index；配置 Embedding 后还会生成向量，并从文档链接或代码引用中提取 Edge。

这些步骤把内容变得容易搜索，但没有替代 Source 的所有权、版本和审核流程。Page、Chunk、Embedding 和 Edge 都应该可以从已知 Source 重新构建。

## 中文文档已经导入，为什么还是搜不到

当前 GBrain MVP 使用 PGLite，初始化参数是：

```bash
gbrain init --pglite --no-embedding
```

这意味着当前实际依赖的是 PostgreSQL Keyword Search，而且 FTS language 默认为 `english`。

我用同一个 Brain 做了一组直接检索：

| Query | Hit |
| --- | ---: |
| `事实源` | 3 |
| `本地知识` | 2 |
| `Revision 是什么` | 0 |
| `Workspace Revision 是什么` | 0 |
| `工作区版本是什么` | 0 |
| `Workspace Revision` | 3 |
| 包含多个英文领域词的完整中文 Prompt | 3 |

这个结果说明它不是“完全不支持中文”。索引里恰好存在的完整中文词可以命中，英文领域词也很稳定；真正不可靠的是中文自然问句。

例如：

```text
Workspace Revision 是什么
```

检索器可能把 `Workspace`、`Revision` 和后面的中文片段一起当作必须满足的条件。只要中文 token 与索引里的形式不一致，整个查询就可能变成零召回。

## `simple` 不是中文分词器

GBrain 支持通过 `GBRAIN_FTS_LANGUAGE` 改变 PostgreSQL text search configuration。看到这里很容易产生一个朴素想法：把 `english` 改成 `simple`，是不是中文就好了？

答案是否定的。

`simple` 的主要作用是取消英文停用词和 stemming，不会自动把：

```text
工作区修订是不可变资产集合快照
```

切成：

```text
工作区 / 修订 / 不可变 / 资产 / 集合 / 快照
```

因此，改成 `simple` 最多改变 token 的处理规则，不能补齐中文 tokenizer。严格意义上的中文 FTS 仍然需要 `jieba`、`zhparser`、Lindera 或其他中文分词能力。

## Embedding 能解决什么，不能解决什么

Embedding 能把问题和文档映射到语义空间，适合处理：

```text
Revision 是什么？
工作区如何固定可信资产？
一次执行怎样保证知识版本一致？
```

这些问题用词不同，但都可能指向 Workspace Revision。

不过，Embedding 不是 FTS。它擅长语义相似，不保证函数名、错误码、配置项、缩写和精确领域词一定排在最前面。对于代码和 AI Infra 项目，完全用 Vector Search 替代关键词检索同样会出问题。

正确结构是 Hybrid Search：

```text
Keyword Search ─┐
                ├→ RRF Fusion → Metadata / Permission Filter → Evidence
Vector Search ──┘
```

Keyword Search 守住精确术语，Vector Search 补足中文和同义表达，再通过 RRF 合并两个排序列表。

## 应用侧的三阶段修复路线

### 第一阶段：Query Rewrite

这是成本最低、应该最先做的一步。

用户输入：

```text
Workspace Revision 是什么？
```

应用侧生成三条确定性 Query：

```text
原始：Workspace Revision 是什么
精确：Workspace Revision
别名：Workspace Revision 修订 版本快照
```

每条分别检索，最后合并去重。别名不应该每次调用 LLM 临时生成，而应该来自项目的 `CONTEXT.md` 和受治理 Glossary，这样结果可解释、可重复。

### 第二阶段：Multilingual Embedding

创建一个带 Embedding 的新 Brain，重新导入全部 Source，再启用 GBrain 已有的 Hybrid Search。

这里不能直接在当前 PGLite 上修改一个配置完事。Embedding dimensions 会影响 Schema，当前 `no-embedding` Brain 应该按照下面的方式迁移：

```text
保留旧 Brain
→ 初始化 Brain v2
→ 重新导入所有 Source
→ 回填 Embedding
→ 跑中文检索评测
→ 验证后原子切换
→ 保留旧 Brain 回滚
```

这也是为什么事实源必须独立于索引存在：只有 Source 可靠，索引才能放心重建。

### 第三阶段：真正的本地中文 FTS

如果后续评测证明 Embedding 仍不能满足精确中文术语检索，再给应用侧增加中文 Side Index：

```text
GBrain Vector Search
+ GBrain Keyword Search
+ 本地 Tantivy 中文 BM25
→ RRF Fusion
```

另一条路线是在云端 PostgreSQL 使用 `pg_jieba` 或 `zhparser`，但它增加了数据库扩展和部署运维复杂度，不适合当前以本地 PGLite 为主的 MVP。

## 不能只看回答“像不像对”

中文检索升级之后，我会准备一组带期望 Citation 的评测数据：

```json
{
  "query": "Workspace Revision 是什么？",
  "expectedSlugs": [
    "example-workspace-spec"
  ]
}
```

至少观察下面几个指标：

| 指标 | 关注的问题 |
| --- | --- |
| Recall@5 | 正确文档是否进入前 5 |
| MRR@5 | 正确文档排得是否足够靠前 |
| Zero-hit Rate | 中文问句是否仍频繁零召回 |
| Citation Precision | 引用是否真的支撑回答 |
| P95 Latency | Hybrid Search 增加了多少延迟 |
| Scope Leakage | 是否检索到其他 Workspace 的知识 |

最后一项尤其重要。一个全局 Brain 如果没有绑定 Workspace 和 Source，再高的召回率也可能意味着更严重的跨项目污染。

## 要不要用 / 我的判断框架

值得现在做：

- 把 Raw Knowledge、GBrain Index 和 Knowledge Asset 分开建模。
- 每个项目按 `team-specs`、`project-repo`、`project-raw` 划分 Source。
- 用 Query Rewrite 立即修复中英混合问句。
- 建立带期望 Citation 的中文评测集。

可以随后做：

- 迁移到 Multilingual Embedding。
- 用 Keyword + Vector + RRF 构建 Hybrid Search。
- 把检索 Trace、Hit Count、Latency 和 Citation 暴露到应用侧 UI。

重点关注：

- 不要把 `GBRAIN_FTS_LANGUAGE=simple` 当成中文 FTS。
- 不要因为文档已经进入 GBrain 就称它为 Knowledge Asset。
- 不要让索引成为无法重建、无法追溯的第二事实源。
- 不要在没有 Workspace / Source Scope 的情况下把多个项目全部塞进默认 Brain。

我现在对这套系统的判断比刚接入 GBrain 时更保守，也更清楚：真正困难的从来不是把文档导进去，而是决定哪些内容可信、什么时候有效、检索为什么命中，以及 Agent 最后引用的到底是哪一条事实。
