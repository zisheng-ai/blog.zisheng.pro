---
title: Evidence 到底是什么：Agent 系统如何证明自己真的完成了任务
date: 2026-08-05 02:15:00
description: Evidence 不是知识、引用或日志的别名，而是 Agent 系统用来支撑或反驳具体 Claim 的可验证事实记录。本文从 W3C PROV、Verifiable Credentials、in-toto、SLSA 与 Agent Tracing 出发，讲清 Evidence 的领域边界、数据模型、生命周期和生产验收方法。
categories:
  - [AI Infra]
tags:
  - Agent
  - Evidence
  - Claim
  - Verification
  - Provenance
  - Evaluation
  - AI Infra
cover: https://blog.zisheng.pro/images/evidence-verification-chain.webp
---

<img src="https://blog.zisheng.pro/images/evidence-verification-chain.webp" data-lazy-src="https://blog.zisheng.pro/images/evidence-verification-chain.webp" alt="Evidence 在 Agent 系统中的验证链路：Source、Evidence、Claim、Evaluation 与 Verification">

*Evidence 不负责让 Agent 看起来更可信。它负责把“相信”变成一件可以复查、反驳和重新计算的事。*

最近我反复看到 `Evidence`。它出现在 LLM Wiki、RAG、Agent 执行、评测、审计和业务验收中。有的产品把检索片段叫 Evidence，有的把截图和日志叫 Evidence，还有的把一整条 Trace 都塞进 Evidence Store。

这些做法都碰到了 Evidence 的一部分，但没有回答最核心的问题：**Evidence 到底在证明什么？**

上一篇[《Agent 系统的一级领域对象：从 Task、Run 到 Evidence》](https://blog.zisheng.pro/posts/ef5159a9b8f5/)把 Evidence 放进了 Agent 的一级领域模型。这一篇只拆 Evidence。我想讲清它为什么属于 Trust & Verification、为什么不能和 Source、Citation、Trace、Artifact、Proof、Evaluation 混在一起，以及进入生产后应该怎样建模。

## 一句话总结

我对 Evidence 的定义是：

> **Evidence 是带有来源、定位、时间、范围和完整性信息，并明确支持或反驳某个 Claim 的可验证事实记录。**

它处在这样一条链路中：

```text
Source / Observation
        ↓ 捕获与固化
Evidence ── supports / refutes ──> Claim
        ↓                               ↓
   Provenance                    Evaluation Policy
        └──────────────> Verification Result
```

这条定义的锚点是“某个 Claim”。单独存在的事实，还没有获得证明方向。

**没有 Claim 的 Evidence，只是 Source、Record 或 Observation。**

## Evidence 属于什么范畴

如果按 Agent 系统的领域边界划分，Evidence 属于：

```text
Trust & Verification
```

它连接三个世界：

| 世界 | 核心问题 | 对象 |
| --- | --- | --- |
| Execution | Agent 做了什么 | Run、Tool Call、Trace |
| Outcome | Agent 交付或声称了什么 | Artifact、Claim |
| Trust | 我们凭什么相信它 | Evidence、Evaluation、Verification Result |

Evidence 不属于 Knowledge Plane。Knowledge、Memory 和 Context 解决“Agent 能看到什么、记住什么”；Evidence 解决“系统凭什么接受 Agent 的某条结论”。

它也不属于 Observability。Trace 和 Log 记录过程，Evidence 从过程记录、交付物或权威系统中挑出与 Claim 有关的最小事实集合。

## 一份材料怎样才成为 Evidence

<img src="https://blog.zisheng.pro/images/evidence-qualification-gates.webp" data-lazy-src="https://blog.zisheng.pro/images/evidence-qualification-gates.webp" alt="一份原始材料通过 Claim、Provenance、Scope 与 Integrity 四道门成为 Evidence">

一份材料不会因为被 Agent 检索到、截图了或者写进数据库，就自动成为 Evidence。它至少要通过四道门。

### 1. Claim Binding：它在证明什么

Evidence 必须绑定具体 Claim，并声明方向：

```text
supports
refutes
```

例如一次 HTTP 200：

- 可以支持“这个 URL 在 02:10 可访问”；
- 不能单独支持“新版本已经发布”；
- 更不能支持“新功能符合业务预期”。

同一条事实对不同 Claim 的证明力完全不同。Evidence 的证明力始终**相对于 Claim**而存在。

### 2. Provenance：它从哪里来

Evidence 必须能够回到产生它的主体、活动和原始实体。

[W3C PROV-O](https://www.w3.org/TR/prov-o/)用 `Agent`、`Activity`、`Entity` 作为 Provenance 的起点：谁承担责任、什么活动发生过、活动使用或生成了什么实体。对 Agent 系统而言，至少要保留：

- Source System；
- Source Object 或 URI；
- Collector Identity；
- Run ID；
- Capture Method；
- Observed At；
- Content Digest。

只有 URL 通常不够。URL 指向的内容可能变化，权限也可能变化。必要时要保存 Snapshot、精确 Locator 和 Hash。

### 3. Scope & Time：它在什么范围内有效

Evidence 必须说明：

- 针对哪个环境；
- 哪个版本或 Revision；
- 哪个账号、租户或业务对象；
- 哪个时间点；
- 哪些前置条件；
- 什么时候过期。

测试环境通过不等于生产环境通过；昨天的库存状态不能证明今天仍有货；管理员账号看到的页面不能证明普通用户也能看到。

### 4. Integrity：它有没有被替换或篡改

Integrity 解决“我现在看到的，还是不是当时采集的那份材料”。常见手段包括：

- 内容 Hash；
- 不可变 Artifact ID；
- 数字签名；
- 时间戳；
- Append-only Ledger；
- 受控的 Source of Truth。

这里要特别区分 Evidence 和 Proof。[W3C Verifiable Credentials Data Model 2.0](https://www.w3.org/TR/vc-data-model-2.0/)明确指出，Evidence 是支持 Credential 中 Claim 的材料；Securing Mechanism 用来验证签发者身份和 Credential 完整性。Credential 能被密码学验证，并不自动意味着其中 Claim 为真。

换到 Agent 领域同样成立：**签名可以证明“这条记录由谁签发、有没有被改”，不能替你判断业务结论是否正确。**

## Evidence 对象内部应该有什么

<img src="https://blog.zisheng.pro/images/evidence-object-anatomy.webp" data-lazy-src="https://blog.zisheng.pro/images/evidence-object-anatomy.webp" alt="Evidence 对象的数据结构：Claim、Source、Locator、Scope、Time、Integrity 与 Provenance">

一个可复用的 Evidence 对象，可以先从下面这组字段开始：

```ts
type Evidence = {
  id: string;
  claimId: string;

  type: 'document_span' | 'system_state' | 'test_result'
    | 'screenshot' | 'artifact_digest' | 'attestation';
  polarity: 'supports' | 'refutes';

  sourceRef: string;
  locator?: string;
  snapshotRef?: string;
  contentDigest?: string;

  scope: {
    environment?: string;
    revision?: string;
    subject?: string;
    tenant?: string;
  };

  observedAt: string;
  validAt?: string;
  expiresAt?: string;

  collectedBy: string;
  runId?: string;
  captureMethod: string;
  integrityStatus: 'unknown' | 'verified' | 'failed';
  status: 'candidate' | 'validated' | 'rejected'
    | 'superseded' | 'expired';
};
```

下面是一套最小领域结构，综合了几类成熟模型的共同思想：

- W3C PROV 的 Agent、Activity、Entity 与派生关系；
- [in-toto Statement](https://github.com/in-toto/attestation/blob/main/spec/v1/statement.md)用 Subject Digest、Predicate Type 和 Predicate 把陈述绑定到不可变对象；
- [SLSA 1.2](https://slsa.dev/spec/v1.2/)用 Provenance Attestation 描述软件产物如何生成，再由消费者执行验证策略；
- Verifiable Credentials 用 Issuer、Subject、Validity、Status、Evidence 和 Securing Mechanism 组织可验证声明。

我不会直接照搬其中任何一个 Schema，因为 Agent 的 Evidence 不只服务软件供应链或 Credential。真正值得复用的是它们的共同原则：**先固定 Subject 与 Claim，再保存 Predicate、Provenance、Scope 和 Integrity。**

## Evidence 和相近概念的边界

<img src="https://blog.zisheng.pro/images/evidence-boundary-map.webp" data-lazy-src="https://blog.zisheng.pro/images/evidence-boundary-map.webp" alt="Source、Citation、Context、Trace、Artifact、Evidence、Proof 与 Evaluation 的边界">

### Source 不是 Evidence

Source 表达信息来自哪里，Evidence 表达 Source 中哪部分事实支撑哪个 Claim。

一份 200 页政策 PDF 是 Source。第 27 页表格中的某个单元格，加上版本、生效日期和定位信息，才可能成为某条 Claim 的 Evidence。

### Citation 不是 Evidence

Citation 是 Claim 指向 Source 或 Evidence 的定位关系。它解决“去哪里看”，不自动解决“这段内容是否真的支撑结论”。

只有链接、没有 Snapshot 和 Locator 的 Citation，还会受到内容漂移影响。

### Context 不是 Evidence

Context 是本次 Run 获准看到的信息边界。材料进入 Context，只能说明 Agent 看过它，不能说明它正确、相关或足以支撑结论。

### Trace 不是 Evidence

[OpenTelemetry](https://opentelemetry.io/docs/specs/otel/trace/api/)把 Trace 组织为由 Span 构成的操作链；[OpenAI Agents SDK Tracing](https://openai.github.io/openai-agents-python/tracing/)会记录 LLM Generation、Tool Call、Handoff、Guardrail 和自定义事件。这些内容非常适合调试、监控和复盘。

但 Trace 回答的是“系统记录到什么过程”，不是“业务结果是否成立”。

```text
Trace: Agent 调用了 deploy()，接口返回 accepted
Claim: 新版本已经在生产环境生效
```

前者不能直接证明后者。只有部署平台状态、线上版本标识和用户可见结果，才能形成后者的 Evidence Bundle。

Trace 当然可以成为 Evidence。例如它能支撑“Agent 在 02:03 调用了某个 Tool”。但它必须绑定这个具体 Claim，并具备可信的采集与完整性边界。

### Artifact 不是 Evidence

Artifact 是 Agent 产出的文件、代码、报告或数据。Artifact 存在，只能证明有交付物；它不能自动证明交付物正确。

测试报告可以同时扮演 Artifact 和 Evidence，但这是两个 Role：作为 Artifact，它是产物；作为 Evidence，它支撑“测试通过”这条 Claim。

### Evaluation 不是 Evidence

Evidence 是事实材料，Evaluation 是依据规则作出的判断。

```text
Evidence + Evaluation Policy → Verification Result
```

不要把模型打出的 `0.87` 直接塞进 Evidence。这个分数属于 Evaluation Result；生成分数所依据的事实、样本和规则才是 Evidence 与 Policy。

## 一个 Coding Agent 的完整例子

<img src="https://blog.zisheng.pro/images/evidence-delivery-ladder.webp" data-lazy-src="https://blog.zisheng.pro/images/evidence-delivery-ladder.webp" alt="Coding Agent 从构建、提交、推送、部署到生产验收的 Claim 与 Evidence 阶梯">

假设 Coding Agent 说：“功能已经发布。”

这句话至少隐藏了五条 Claim：

| Claim | 可以采用的 Evidence | 不能替代它的材料 |
| --- | --- | --- |
| 代码已修改 | 目标范围内的 Git Diff | Agent 的自然语言总结 |
| 修改通过工程验证 | 测试结果、Build Artifact、退出码 | 文件已经保存 |
| 版本已提交 | Commit SHA、Tree Hash | 本地工作区状态 |
| 版本已推送 | Remote Ref 指向目标 Commit | 本地 Commit 存在 |
| 版本已部署 | Deployment ID、目标环境状态 | Push 成功 |
| 生产结果符合预期 | 生产 URL、版本指纹、状态检查、真实页面或 API 结果 | Deployment Job 显示 success |

这里最容易犯的错误，是拿低一层 Evidence 支撑高一层 Claim：

```text
Build passed  ≠  Deployed
Git pushed    ≠  Production updated
HTTP 200      ≠  Expected version served
Screenshot    ≠  All users can complete the flow
```

Evidence-based Delivery 应先把大而模糊的“完成”拆成可验证 Claim，再为每条 Claim 选择最接近 Source of Truth 的 Evidence。继续堆积截图和日志不会自动补上这层关系。

## 在 RAG 与 LLM Wiki 里，Evidence 应该是什么

假设回答中出现一条结论：

```text
Claim：订单支付后 24 小时内可以免费取消。
```

一个完整的 Evidence 至少应该包含：

```text
Source:        cancellation-policy.pdf
Revision:      2026-07-18
Locator:       page=12, table=2, row=3
Quoted Span:   支付完成后 24 小时内……
Effective At:  2026-07-20
Scope:         普通订单；不含特价产品
Digest:        sha256:...
Polarity:      supports
```

向量检索返回的 Chunk 只能算 Candidate Evidence。它还可能：

- 命中过期版本；
- 丢失表头或脚注；
- 越过权限范围；
- 只在相似语义上相关；
- 无法支持回答中的具体数字。

因此 LLM Wiki 真正应该管理的，不只是 Chunk 和 Citation，而是：

```text
Claim → Evidence Span → Source Revision → Permission Scope
```

这时用户点开引用，看到的不只是“模型参考了哪篇文档”，而是“这句话具体由哪一段、哪个版本、在什么范围内支撑”。

## 在 GUI Agent 里，点击不是 Evidence

GUI Agent 的 Trace 经常是：

```text
点击“提交”
等待 2 秒
页面出现绿色提示
```

这只能证明执行过程被记录。业务验收应根据 Claim 选择更直接的 Evidence：

- 表单提交：服务端返回的业务 ID；
- 订单创建：订单系统中的新记录；
- 退款完成：支付系统状态与交易号；
- 文件上传：对象存储中的 Key、Hash 与读取验证；
- 页面发布：生产 URL 的版本指纹与用户可见内容。

截图可以作为补充 Evidence，但它通常存在三个限制：难以结构化比对、可能截错环境、无法证明后端状态。只要存在权威业务系统，就应该优先从权威系统重新读取状态。

## Evidence 的生命周期

Evidence 不是写入后永久有效的附件。它应该有自己的生命周期：

```text
candidate
  → captured
  → validated
  → admitted / rejected
  → superseded / expired / revoked
```

这里有两层判断，不能混在一起：

1. **Evidence Validation**：材料是否真实、完整、可定位、未过期，能否进入证据集合；
2. **Claim Evaluation**：证据集合是否足以让某条 Claim 通过、失败或保持不确定。

可能出现这些情况：

- Evidence 本身有效，但与 Claim 无关；
- 两条有效 Evidence 相互冲突；
- Evidence 支持 Claim，但覆盖范围不足；
- 原本有效的 Evidence 因 Source Revision 更新而过期；
- 新 Evidence 推翻了旧 Verification Result。

所以 Verification Result 最好也是可重算的派生对象，而不是直接写死在 Evidence 上。

## 什么时候应该把 Evidence 做成一级对象

简单聊天机器人可以把 Citation 和 Evidence Span 放在 Message Metadata 中，不必一开始建设复杂平台。

当出现下面任意几种需求时，Evidence 应升级为一级领域对象：

- 同一条 Evidence 会被多个 Claim、Run 或 Evaluator 引用；
- Evidence 需要独立权限与脱敏；
- Evidence 会过期、撤销、被替代或重新验证；
- 系统需要审计“当时为什么通过”；
- 需要比较不同 Agent、模型或版本采用了哪些证据；
- 需要第三方复查，或者生成 Acceptance Receipt；
- 业务不能接受 Agent 用自己的输出证明自己。

最小架构可以分成五块：

```text
Evidence Collector   从 Trace、Artifact、Source of Truth 捕获候选材料
Evidence Normalizer  补齐 Locator、Scope、Time、Digest、Provenance
Evidence Store       保存不可变 Snapshot 与关系
Claim Verifier       按 Policy 组合和评估 Evidence
Verification Ledger  记录谁在何时基于哪组 Evidence 作出什么判断
```

## 常见反模式

### 把所有运行数据都叫 Evidence

结果是 Evidence Store 退化成另一个日志平台。数据很多，但无法回答任何具体 Claim。

### 让 Agent 自己证明自己

Agent 输出“工具调用成功”只能算自述。高风险 Claim 应由独立 Verifier 从权威系统重新读取状态。

### 只保存链接，不保存版本和定位

链接内容一变，历史 Verification Result 就无法复现。关键材料至少要有 Revision、Locator 或 Digest。

### 一个 Screenshot 证明整条业务链

截图能证明某个时刻的视觉状态，不能自动证明后端数据、权限覆盖、持久化和其他用户视角。

### 把 Confidence 写在 Evidence 上

Confidence 依赖 Claim、Evidence Bundle、Evaluator 和 Policy。把一个全局置信度塞给单条 Evidence，会掩盖判断条件。

### 只保留通过的 Evidence

反驳材料、冲突材料和被拒绝原因同样重要。否则系统会形成只为既有结论找支持的确认偏误。

## 我的判断框架

当一个系统说“我们有 Evidence”时，我会连续问八个问题：

1. 它绑定的 Claim 是什么？
2. 它是支持还是反驳 Claim？
3. Source of Truth 是谁？
4. 能否精确定位并重新获取原始事实？
5. 它适用于哪个环境、版本、主体和时间？
6. 如何确认内容没有被替换？
7. 谁负责验证，依据哪条 Policy？
8. 出现冲突、过期或撤销后，旧结论能否重新计算？

如果这些问题回答不出来，系统拥有的多半只是 Logs、Links、Screenshots 或 Retrieved Chunks，还没有真正建立 Evidence Model。

Evidence 对 Agent 产品最重要的价值，是把“完成”从一次自述改造成可以独立验收的结论：

> Agent 可以提出 Claim，但不能只靠自己的话完成验收。系统必须保留足够的 Evidence，让另一个人、另一个 Verifier，甚至未来的另一套模型能够重新得到或推翻这个结论。

## 参考索引

以下均为本文实际采用的规范、标准或官方工程文档，检索日期为 2026-08-05。

| 范畴 | 参考资料 | 本文采用的依据 |
| --- | --- | --- |
| Provenance | [W3C PROV-O：The PROV Ontology](https://www.w3.org/TR/prov-o/) | Agent、Activity、Entity，以及生成、使用、归属、派生和失效关系 |
| Claims / Evidence | [W3C Verifiable Credentials Data Model 2.0](https://www.w3.org/TR/vc-data-model-2.0/) | Evidence、Issuer、Subject、Validity、Status 与 Securing Mechanism 的边界；可验证不等于 Claim 为真 |
| Attestation | [in-toto Statement Layer Specification](https://github.com/in-toto/attestation/blob/main/spec/v1/statement.md) | Subject Digest、Predicate Type、Predicate 的绑定结构 |
| Software Provenance | [SLSA Specification 1.2](https://slsa.dev/spec/v1.2/) | Provenance Attestation、Verification Summary 与消费者验证策略 |
| Observability | [OpenTelemetry Tracing API](https://opentelemetry.io/docs/specs/otel/trace/api/) | Trace、Span、Event、Status 的过程记录模型 |
| Agent Runtime | [OpenAI Agents SDK：Tracing](https://openai.github.io/openai-agents-python/tracing/) | Agent Run 中 Generation、Tool Call、Handoff、Guardrail 等 Trace 数据的范围 |
| AI Risk | [NIST AI RMF：Generative AI Profile](https://www.nist.gov/publications/artificial-intelligence-risk-management-framework-generative-artificial-intelligence) | Governance、Pre-deployment Testing、Content Provenance 与 Incident Disclosure 的风险管理视角 |

本文给出的 Evidence 定义、生命周期和字段模型，是基于这些资料进行的 Agent 领域建模推导，不是任何单一规范的原文 Schema。
