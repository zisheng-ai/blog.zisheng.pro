---
title: Claude Fable 5 发布：有哪些大更新？
date: 2026-07-03 12:00:00
description: Anthropic 发布了目前能力最强的正式版模型 Claude Fable 5。这篇文章从能力升级、规格定价、API 破坏性变更和 Prompting 风格四个维度，梳理它相比 Opus 4.8 的所有大更新，帮你判断要不要升级、怎么迁移。
categories:
  - [AI]
tags:
  - Claude
  - Anthropic
  - Fable 5
  - LLM
  - Agent
---

Anthropic 发布了 **Claude Fable 5**（model ID：`claude-fable-5`），这是目前 Anthropic 能力最强的正式发布模型，定位是"最苛刻的推理任务和长程 agentic 工作"。我第一时间把 Claude Code 的默认模型切换了过去，这篇文章梳理一下它相比 Opus 4.8 的大更新，以及写代码接入时要注意的破坏性变更。

## 一句话总结

Fable 5 的核心卖点不是"老任务做得更好"，而是**能做以前模型做不了的事**：单次请求自主运行几十分钟的长程任务、可靠的多 sub-agent 并行协作、端到端的企业级交付物。如果你的场景还是分类、摘要、单轮问答，Opus 4.8 依然是更划算的选择。

## 能力升级

### 长程 agentic 任务（最大亮点）

单次请求可以自主运行很长时间——复杂任务跑 15 分钟是正常现象。适合的场景包括：

- 大型代码迁移、重构、通宵编码任务
- 端到端的企业交付物：财务分析、Excel 表格、PPT、文档
- 深度研究和长时间的自主工作流

官方给的最佳实践是：**在第一轮就给出完整、清晰的任务规格**，然后放手让它执行，而不是挤牙膏式地多轮补充需求。

### 多 sub-agent 协作

以前的模型用 sub-agent 经常翻车，很多团队的 prompt 里都有"不要随便开 sub-agent"之类的压制性指令。Fable 5 反过来了：**并行委派子任务非常可靠**，能和长期运行的 sub-agent、peer agent 保持异步通信。官方甚至建议主动鼓励它委派：

> Delegate independent subtasks to sub-agents and keep working while they run.

### 其他提升

- **Code review / debugging**：找真实 bug 的能力显著提升（recall 和 precision 都涨了）
- **Vision**：针对密集、模糊、翻转、有噪点的图像做了专门训练，会主动用 bash 和 crop 工具预处理图片
- **Memory**：给它一个可写的 memory 文件（哪怕就是个 `.md`），跨任务表现明显更好

## 规格与定价

| 项目 | Claude Fable 5 | Claude Opus 4.8 |
| --- | --- | --- |
| Context window | 1M tokens（默认即最大） | 1M tokens |
| Max output | 128K | 128K |
| 输入价格 | $10 / MTok | $5 / MTok |
| 输出价格 | $50 / MTok | $25 / MTok |
| Tokenizer | 与 Opus 4.8 相同 | — |

价格是 Opus 4.8 的两倍。Tokenizer 没变，从 Opus 4.7/4.8 迁移过来 token 数基本不变，只是单价不同。

## API 破坏性变更

写代码接入时要注意这几点，都是会直接 400 的硬约束：

### 1. Thinking 永远开启

不要传 `thinking` 参数（或只传 `{"type": "adaptive"}`）。`{"type": "disabled"}` 和 `budget_tokens` 都会返回 400。思考深度改用 `output_config.effort` 控制，档位从 `low` 到 `max`：

```python
client.messages.create(
    model="claude-fable-5",
    max_tokens=16000,
    output_config={"effort": "high"},  # 不传 thinking 参数
    messages=[...],
)
```

值得一提的是，Fable 5 的 `low` 档表现经常超过旧模型的 `max` 档，日常任务不用无脑拉满。

### 2. 原始思维链永不返回

默认 `display: "omitted"`，thinking 块的内容是空字符串。想看推理摘要要显式设置：

```python
thinking={"type": "adaptive", "display": "summarized"}
```

### 3. 安全分类器与 refusal fallback

Fable 5 对生物研究和网络安全内容有前置安全分类器，被拒的请求返回 HTTP 200 + `stop_reason: "refusal"`。问题是**良性的相邻工作也可能被误伤**（比如安全工具开发、生命科学任务），所以官方建议默认开启 server-side fallbacks——被拒时同一个请求内自动降级到 Opus 4.8 重跑：

```python
response = client.beta.messages.create(
    model="claude-fable-5",
    max_tokens=16000,
    betas=["server-side-fallback-2026-06-01"],
    fallbacks=[{"model": "claude-opus-4-8"}],
    messages=[...],
)
```

### 4. 其他硬约束

- **不支持 assistant prefill**（和 4.6+ 系列一致），用 `output_config.format` 结构化输出替代
- **Sampling 参数全部移除**：`temperature` / `top_p` / `top_k` 都会 400
- **要求 30 天数据保留**：ZDR（零数据保留）组织的所有请求直接 400，排查时别只盯着请求体

## Prompting 风格变化

这是迁移时最容易被忽略的一点：**为旧模型写的 prompt 往往过于 prescriptive，反而会降低 Fable 5 的输出质量**。

旧模型需要你把步骤掰开揉碎写清楚，Fable 5 更适合只描述目标和约束，把"怎么做"留给它自己规划。官方建议迁移后做一轮 A/B 测试，把老的 step-by-step 脚手架删掉对比效果。

另外由于单轮任务可能跑很多分钟，工程上要提前规划好 timeout、streaming 和进度展示，把工作流设计成异步 check-in 的模式，而不是阻塞在一个请求里干等。

## 要不要升级？

我的判断框架：

- **值得上 Fable 5**：长程自主任务、多 agent 编排、复杂重构、端到端交付物——这些是它独有的能力区间
- **留在 Opus 4.8**：常规编码、日常问答、成本敏感的高频调用——一半的价格，能力足够
- **留在 Sonnet 4.6 / Haiku 4.5**：分类、摘要、大批量的简单任务

对 Claude Code 用户来说，切到 Fable 5 最直观的感受是：单轮跑得更久但更完整，更愿意开 sub-agent 并行干活，最终总结更清晰。偶尔碰到 refusal 误伤时换个表述或临时降回 Opus 4.8 即可。
