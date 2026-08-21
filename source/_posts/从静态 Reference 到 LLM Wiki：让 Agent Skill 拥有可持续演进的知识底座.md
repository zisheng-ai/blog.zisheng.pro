---
title: 从静态 Reference 到 LLM Wiki：让 Agent Skill 拥有可持续演进的知识底座
date: 2026-08-21 11:22:19
description: Agent Skill 可以封装流程，却不适合长期内嵌持续变化的业务知识。本文以脱敏的数据研发场景为例，拆解 Raw Sources、Wiki、Schema 三层架构，以及 Ingest、Query、Lint 如何把静态 Reference 升级为可追溯、可校准、可持续演进的知识底座。
categories:
  - [AI]
tags:
  - Agent
  - Skill
  - LLM Wiki
  - Knowledge Engineering
  - RAG
  - AI Coding
cover: /images/agent-skill-llm-wiki-knowledge-loop.webp
---

![Agent Skill 从静态 Reference 到 LLM Wiki 的知识架构](/images/agent-skill-llm-wiki-knowledge-loop.webp)

最近我读到一篇数据研发实践文章。它讨论了一个很具体的问题：团队已经把需求调研、模型设计、代码生成和任务发布封装成 Agent Skill，但模型口径、数据资产和业务知识仍在持续变化，静态写进 `Reference` 的内容很快就会落后。

我认为这篇实践最值得复用的部分，不是某个知识库产品的接入方式，而是它重新划清了 Skill 与知识的边界：**Skill 负责稳定的工作方法，知识库负责持续变化的事实。**

> 说明：本文是脱敏后的原创技术解读。公开版未复用原文、内部链接、组织与人员信息、真实业务数据及操作截图，仅保留可泛化的工程方法。

<!-- more -->

## 一句话总结

当 Agent Skill 依赖的知识规模变大、变化变快之后，不要继续把资料塞进 `Reference`。更稳妥的结构是：

```text
Raw Sources  →  Wiki  →  Schema / Skill
事实源           知识编译     查询与执行规则
```

Agent 通过 `Ingest` 把新资料编译进 Wiki，通过 `Query` 为当前任务取得带来源的知识，通过 `Lint` 发现冲突、孤岛和过期内容。每次实践产生的新结论再回到 Raw Sources，进入下一轮增量编译。

## 静态 Reference 为什么会失效

第一版 Skill 把领域知识放进 `references/` 很合理。它简单、透明、可以和代码一起版本化，也不需要额外基础设施。

问题出现在知识开始具备下面几个特征之后：

- 数据表、字段口径和任务代码持续变化；
- 需求文档、设计文档、规范和历史案例分散在不同位置；
- 同一个知识点可能存在多个版本，甚至相互冲突；
- 一次任务中校准出的结论，需要被下一次任务继续使用；
- 不同团队成员知道的信息不对称，Agent 也无法判断哪份资料更新。

如果继续扩充静态 Reference，Skill 会逐渐变成一个难以维护的资料压缩包。知识更新需要修改 Skill，Skill 发布又和知识变更绑定；资料越多，Agent 每次加载的 Context 越大，真正需要的内容反而更难命中。

这时需要拆开两个生命周期：

| 对象 | 适合承载什么 | 变化频率 | 维护方式 |
| --- | --- | --- | --- |
| Skill | 流程、输入输出、工具调用、质量门槛、失败恢复 | 相对稳定 | 代码评审与版本发布 |
| Reference | 稳定规则、模板、固定术语、少量示例 | 低频 | 随 Skill 维护 |
| Wiki | 业务知识、数据资产、设计结论、案例经验、关系与冲突 | 持续变化 | 增量编译与健康检查 |
| Raw Sources | 原始文档、代码、规范和已审核事实 | 按业务变化 | 由知识 Owner 维护 |

## LLM Wiki 的三层架构

[Andrej Karpathy 提出的 LLM Wiki pattern](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) 把知识系统分成 Raw Sources、Wiki 和 Schema 三层。这个分层很适合 Agent Skill，因为它同时回答了三个问题：事实在哪里、知识怎样积累、Agent 按什么规则工作。

### Raw Sources：不可被 AI 改写的事实源

Raw Sources 保存原始资料。放到数据研发场景里，可以包括：

- 经过授权的代码仓库与 SQL；
- 数据资产说明、字段口径和模型设计文档；
- 需求调研、业务规则与研发规范；
- 已审核的故障案例、问答记录和设计反馈。

这一层的核心约束是：AI 可以读取，但不能把自己的总结反向覆盖原始事实。任何 Wiki 页面和 Agent 答案都应该能够回到具体 Source、版本和位置。

### Wiki：由 AI 维护的知识中间层

Wiki 不是原文切片的集合。它把分散资料整理成主题页、实体页、流程页、索引、交叉引用和编译日志。

例如，一个“核心交易事实表”主题页可以连接字段口径、上游来源、下游任务、历史变更与常见异常；当新设计文档进入系统时，Agent 不只新增一个 Chunk，还要更新已有页面并标记冲突。

这就是“编译”的价值：跨文档综合只做一次，后续任务可以复用已经整理过的知识，同时在关键结论上回查原文。

### Schema：让 Agent 按领域规则维护知识

Schema 可以写在 `AGENTS.md`、Skill 或专用配置里。它需要定义：

- Wiki 有哪些页面类型和字段；
- 新资料如何进入，哪些页面必须更新；
- Query 时先查什么，什么时候必须回源；
- 哪些结论可以自动写入，哪些必须人工审核；
- 如何记录 Citation、Revision、冲突和不确定性；
- Lint 要检查哪些结构与内容问题。

没有 Schema，LLM Wiki 容易退化成模型自由发挥的笔记目录。Schema 才是把通用 LLM 变成领域知识维护者的关键约束。

## Ingest、Query、Lint：知识库的三个基本操作

三层结构落地后，Agent 只需要掌握三个稳定操作。

### Ingest：把新资料编译进现有知识

一次 Ingest 不应该只生成摘要。完整流程至少包括：

```text
读取新 Source
  → 提取事实、实体、关系与适用范围
  → 查找已有相关页面
  → 新增或更新 Wiki 页面
  → 标记冲突与待确认项
  → 更新 index.md 与 log.md
  → 保留 Source Citation
```

增量更新比全量重建更重要。数据资产和业务口径每天都可能变化，系统必须知道这次新增了什么、替换了什么、哪些旧结论仍然有效。

### Query：先复用知识，再回到事实源

Query 的默认路径是先读取 Wiki，快速定位主题、实体和已有综合结论。遇到下面几类内容时，再回到 Raw Sources 核验：

- 字段口径、时间范围、过滤条件等精确事实；
- 可能已被新版本替代的结论；
- 会影响代码生成或生产任务的关键规则；
- Wiki 中明确标记为冲突或低置信度的内容。

这样既避免每次从海量原文重新综合，也不会把 AI 生成的 Wiki 当成最终事实。

### Lint：让知识库能够长期维护

知识库规模上来以后，检索能否返回答案只是最低门槛。Lint 还要持续检查：

- 同一概念是否出现多个命名；
- 新旧页面是否存在未处理的矛盾；
- 页面是否缺少来源或 Revision；
- 是否出现没有入链的孤岛页面；
- 重要实体是否只被提及，却没有独立页面；
- Source 已更新，Wiki 页面是否仍停留在旧版本。

结构检查可以自动执行；涉及业务语义的冲突，应该进入人工审核，而不是让模型自行决定哪一方是真相。

## Agent Skill 应该怎样改造

改造前，Skill 的知识路径通常是：

```text
用户需求
  → Skill
  → 静态 Reference
  → 生成设计与代码
```

改造后，Skill 不再携带全部业务知识，而是声明查询策略和质量门槛：

```text
用户需求
  → Skill 解析任务
  → Query Wiki 定位知识
  → 回查 Raw Sources 核验关键事实
  → 生成设计与代码
  → 人工校准与执行验证
  → 经验写回 Raw Sources
  → 增量 Ingest
```

Skill 内部可以保留这样的稳定契约：

```yaml
knowledge_policy:
  query_first: wiki
  source_of_truth: raw_sources
  require_citation_for:
    - field_definition
    - business_rule
    - production_code
  reject_when:
    - unresolved_conflict
    - missing_revision
    - permission_denied
  feedback:
    destination: reviewed_case_notes
    ingest_after_review: true
```

这里最重要的变化，是 Skill 从“知识容器”变成“知识消费者和执行编排器”。领域知识可以独立演进，Skill 只需要保持操作协议稳定。

## 一个脱敏的数据研发例子

假设数据同学收到一份“实验分组回收”的需求。任务需要找到现有核心表、确认分组口径、设计新增字段，并生成 DDL 与 ETL 任务。

静态 Reference 只能提供通用建模规范和代码模板。真正决定方案正确性的知识分散在资产说明、历史设计、现有 SQL 与最近一次口径调整中。

接入 LLM Wiki 后，执行链路可以变成：

1. Skill 读取需求调研文档，提取业务对象、指标、时间范围和待确认项。
2. Query 从 Wiki 定位相关主题页、核心表、字段口径和历史设计结论。
3. 对会进入代码的字段与过滤条件回查 Raw Sources，固定 Citation 与 Revision。
4. Skill 生成模型设计文档，显式列出已确认事实、推导结论和待确认问题。
5. 设计确认后，代码生成 Skill 按模板产生 DDL、ETL 与发布清单。
6. 测试和评审中发现的新口径，先进入经过审核的案例记录，再触发增量 Ingest。

这个流程没有让 Agent 自动拥有最终决定权。它缩短的是查资料、建立关联和维护知识的时间；模型设计是否正确，仍然要由 Citation、测试结果和领域 Owner 的确认来证明。

## 实践反哺为什么是关键闭环

很多知识库只完成“资料进入系统”，没有处理“任务执行后产生的新知识”。结果是 Wiki 看起来越来越大，Agent 却在重复踩同样的坑。

真正有价值的反馈通常来自三类场景：

- Agent 找到了资料，但选错了适用范围；
- 设计结论基本正确，但遗漏了一个业务例外；
- 代码运行成功，却暴露出文档和真实实现不一致。

这些反馈不能直接写成新的事实。更稳妥的链路是：

```text
执行观察
  → 人工确认
  → 形成审核后的案例或口径变更
  → 写入 Raw Sources
  → Ingest 更新 Wiki
  → Lint 检查受影响页面
```

这样，知识库会随着真实任务逐步校准，而不是随着模型回答数量机械膨胀。

## 这套架构的四个硬门槛

### 1. Wiki 不能替代事实源

Wiki 是可重建的知识产物。删除 Wiki 后，系统应该能够从 Raw Sources 重新编译；删除 Raw Sources 后，只剩 Wiki 就失去了可信根基。

### 2. Citation 与 Revision 必须进入查询协议

“回答里带了一个链接”还不够。关键 Claim 应该对应具体 Source Span，并固定查询时使用的 Revision。否则新旧口径仍可能混进同一个方案。

### 3. 权限必须早于召回

知识编译不能绕过原始权限。Source、Wiki 页面和查询结果都要继承 Scope 与 Permission，过滤应该发生在召回和生成之前。

### 4. 评测不能只看代码生成比例

AI 生成了多少代码，只能说明工具参与度。知识底座更应该评估：核心事实召回率、Citation 正确率、冲突发现率、过期知识命中率、方案返工率，以及任务完成后是否形成了可复用的新知识。

## 要不要用

我的判断是：

- **值得上**：同一类任务反复发生，知识分散且持续变化，错误口径会直接影响设计或生产代码。
- **可以再等**：资料规模小、规则稳定、任务复用率低，几份静态 Reference 已经能够清楚覆盖。
- **重点关注**：编译产物的 Citation 完整性、增量更新的影响范围、语义冲突的人工审核，以及权限是否贯穿 Raw、Wiki 与 Query。

不要把 LLM Wiki 当成更复杂的 RAG，也不要把它当成新的万能知识平台。它最有价值的地方，是把 Agent 每次临时找资料、临时综合、临时纠错的过程，沉淀成一个可以持续维护的知识系统。

对 Agent Skill 来说，合理的边界也因此更清楚：**稳定方法写进 Skill，稳定规则留在 Reference，变化中的事实进入知识库，执行中得到的经验经过审核后再反哺事实源。**

## 参考资料

- [Andrej Karpathy：LLM Wiki](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f)

> 本文使用 [writting-skill](https://github.com/zisheng-ai/writting-skill) 辅助写作，配图使用 [better-imagegen](https://github.com/zisheng-ai/better-imagegen) 生成。项目已开源，欢迎在 GitHub 点个 Star。
