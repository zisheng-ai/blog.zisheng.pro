---
title: Claude Code Skill Token 优化实践：从 162KB 到 117KB
date: 2026-07-04 22:00:00
description: 记录一次给 Claude Code 自定义 Skill 做 token 瘦身的实践。核心方法是把 reference 文件按"谁在哪个 phase 用"拆分，把同一个合规说明的三份副本压缩成一份，把 21 个 costume 模板压缩为 5 个 exemplar。结果：最重的单个 reference 文件从 162KB 降到 117KB，每次插图生成额外省 27KB 上下文。适合正在构建 Claude Code 自定义 Skill 的开发者。
categories:
  - [AI]
tags:
  - Claude Code
  - Prompt Engineering
  - AI Infra
  - Token Optimization
cover: /images/claude-code-skill-token-optimization.webp
---

给 Claude Code 写自定义 Skill 已经有一段时间了。最近做了一次专项 token 优化，顺手记录一下思路和方法。

## 一句话总结

Skill 的 reference 文件不是文档，是 context。每 KB 都是运行时成本。按 phase 拆分文件比压缩内容更有杠杆。

---

## 背景：Skill 的 token 消耗来自哪里

Claude Code 的 Skill 系统工作原理很简单：用户触发 Skill → `SKILL.md` 进入 context → 执行时按 phase 按需加载 `references/*.md`。

理论上"按需加载"已经很省了。但当某个 reference 文件本身就是 162KB，问题就来了：

- 它被加载时就是 162KB 的 context 消耗
- 如果这个文件里有一半内容是"当前 phase 不需要的"，那一半就是纯浪费

我的 `cover-allure-elements.md` 就是这个情况。它同时服务两个 phase：
- **A2**（封面生成）：需要 allure 规则 + costume 模板 + genre prompt 模板
- **A2.5**（插图生成）：只需要 allure 规则 + pose 表 + exposure tier，**不需要** genre 完整 prompt 模板（~30KB）

---

## 三类优化，效果从高到低

### 1. 结构性拆分（最高杠杆）

把"不同 phase 的消费者用不同部分"升级为"不同 phase 加载不同文件"。

**操作**：把 `cover-allure-elements.md` 里的 15 个 genre prompt 模板（约 380 行、30KB）拆出来，新建 `cover-genre-playbook.md`。

**SKILL.md reference loading 更新**：
```
A2：加载 cover-allure-elements.md + cover-genre-playbook.md
A2.5：只加载 cover-allure-elements.md
```

**效果**：每次插图生成任务（A2.5）比之前少加载 27KB。这是结构性节省，不是内容压缩——文件总量没变，但每次调用用到的更少了。

### 2. 跨文件去重（中等杠杆）

审查发现同一段合规说明出现了三次：

| 位置 | 内容 | 行数 |
|---|---|---|
| `SKILL.md` | "no nipples/genitals/sex acts" | ~8 行 |
| `cover-allure-elements.md §0` | 同上，散文版 | ~12 行 |
| `adsense-arbitrage.md §1.1` | 同上，加业务背景 | ~10 行 |

处理方式：`cover-allure-elements.md §0` 保留权威版（因为它被 A2 和 A2.5 都加载，离使用者最近），另外两处改为 1 行 cross-reference：

```
§0 floor: per `cover-allure-elements.md §0` — no explicit content. Suggestive allure fully allowed; push it.
```

节省约 22 行。规则只在一个地方维护，更新时不会出现三处不同步的问题。

### 3. 模板压缩（中低杠杆）

`cover-allure-elements.md` 里有 21 个 costume 类型，每个都是完整的 prompt 模板（visual + best-for + prompt block）。问题在于：模型其实只需要看懂"如何构造 costume 提示词"的方法论，不需要 21 份完整示例。

操作：保留 5 个最高频的 costume（Academy、Nurse、Silk Robe、Evening Gown、Wet White Shirt），剩余 16 个改为一行列举：

> Other costume types (same structure — use Fabric Allure Ranking to construct prompts): Flight Attendant, Police, Maid, Qipao, Wedding...

节省约 200 行。模型用 5 个 exemplar 学会方法论，遇到其他类型自行套用。

---

## 工具：如何快速找到可优化点

用一个分析 agent 扫描所有 reference 文件，输出：

1. **跨文件冗余**：同一内容在哪些文件里出现了多次
2. **verbose prose 转 table**：段落式描述是否可以压缩为表格
3. **example 密度**：8 个完整示例是否可以替换为 3 个 exemplar + 方法论
4. **重复 preamble**：每个文件开头的"为什么这很重要"是否是多余重复

Prompt 给到 agent 时要求输出具体行号和节省估算，否则结果太泛化没法执行。

---

## 一个反直觉的结论

**内容压缩是锦上添花，结构分离才是关键。**

文件从 162KB 压缩到 117KB，感觉很好。但真正节省大的是"A2.5 不再加载 genre playbook"那 27KB——它每次插图任务都生效，是乘法效应。

类比到普通工程：代码优化一个函数（内容压缩）vs 拆出一个只在特定路径加载的模块（结构分离）。后者的效益通常更高，因为它减少的是每次调用的基础成本。

---

## 顺带发现的合规问题

`facebook-ads.md` 全文混用了中文术语（广告系列、广告组、类似受众等）。Skill 文件有一条硬规则：必须用英文写。这不是审美问题，是一致性问题——如果 skill 文件被 agent 加载并转述给另一个 agent，中英混杂会增加解析摩擦。

处理方式是直接 find-replace 全部术语：广告系列 → Campaign，广告组 → Ad Set，类似受众 → Lookalike Audience，等。

---

## 要不要做 / 判断框架

**值得做**：

- Skill 被高频调用（每次写章节、每次生图都触发）
- 单个 reference 文件超过 50KB
- 同一个文件被不同 phase 消费，但 phase 之间需要的内容有明显差异

**可以再等**：

- Skill 调用频率低（每月几次）
- 所有 reference 文件都在 20KB 以下
- 文件内容高度内聚，没有明显的 phase 分离机会

**不值得做**：

- 为了"显得更整洁"而拆分（增加维护成本，收益低）
- 把所有内容压到极限（模型需要足够的示例才能理解方法论，过度压缩反而降质量）

---

当前 fiction-site-builder skill 的总 reference 文件大约 830KB，按 phase 加载，单次任务实际加载通常在 100–200KB 区间。优化之后高频 phase（A2.5）少了 27KB，低频 phase（A2）靠新的文件拆分仍然能拿到完整 genre 库。

整体感受：token 优化不是"写少一点"，而是"用对粒度"。
